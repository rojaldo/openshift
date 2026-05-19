# OpenShift Container Platform - Día 2
## Orquestación Avanzada en OpenShift

---

## 2.1 Networking

### Modelo de red de pods: OVN-Kubernetes en OpenShift

OpenShift usa **OVN-Kubernetes** como plugin de red (CNI) por defecto, que implementa una red overlay con Open vSwitch.

**Características del modelo de red:**
- Cada Pod tiene su propia dirección IP
- Los Pods pueden comunicarse entre sí sin NAT
- Los nodos pueden comunicarse con todos los Pods sin NAT
- La IP que un Pod ve es la misma que los demás ven

**OVN-Kubernetes proporciona:**
- Red overlay con VXLAN/Geneve
- NetworkPolicies avanzadas
- IPsec para cifrado de tráfico entre nodos
- Egress IP para tráfico saliente predecible
- Soporte para múltiples redes (Multus)

**Arquitectura OVN-Kubernetes:**

```
┌─────────────────────────────────────────────────────┐
│                   Master Nodes                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │
│  │ ovnkube-    │  │ ovnkube-    │  │ ovnkube-    │ │
│  │ master      │  │ master      │  │ master      │ │
│  └─────────────┘  └─────────────┘  └─────────────┘ │
│         │                │                │         │
│         └────────────────┴────────────────┘         │
│                          │                          │
│                   OVN Northbound                    │
│                    Database                         │
└─────────────────────────────────────────────────────┘
                           │
        ┌──────────────────┴──────────────────┐
        │                                     │
┌───────┴───────┐                    ┌───────┴───────┐
│  Worker Node  │                    │  Worker Node  │
│ ┌───────────┐ │                    │ ┌───────────┐ │
│ │ ovnkube-  │ │                    │ │ ovnkube-  │ │
│ │ node      │ │                    │ │ node      │ │
│ └───────────┘ │                    │ └───────────┘ │
│ ┌───────────┐ │                    │ ┌───────────┐ │
│ │ Pod A     │ │                    │ │ Pod B     │ │
│ │ 10.128.1.2│◄├────────────────────┼─┤ 10.129.1.5│ │
│ └───────────┘ │    (VXLAN overlay) │ └───────────┘ │
└───────────────┘                    └───────────────┘
```

**Ejemplo - Explorar la red del clúster:**

```bash
# Ver configuración de red del clúster
oc get network.config/cluster -o yaml

# Ver pods de red
oc get pods -n openshift-ovn-kubernetes

# Ver configuración de nodo específico
oc get nodenetworkconfigurationenclave -A

# Inspeccionar un Pod de red
oc exec -n openshift-ovn-kubernetes -c ovnkube-node \
  $(oc get pods -n openshift-ovn-kubernetes -l app=ovnkube-node \
  -o jsonpath='{.items[0].metadata.name}') -- ovn-nbctl show

# Ver rangos de IP asignados
oc get network-attachment-definitions -A
oc get ipaddresspools -A
```

---

### DNS interno del clúster

OpenShift corre CoreDNS como servidor DNS interno del clúster.

**Nomenclatura DNS:**
```
<servicio>.<namespace>.svc.cluster.local
```

**Ejemplos:**
- `webapp` → servicio en el mismo namespace
- `webapp.mi-proyecto` → servicio en otro namespace
- `webapp.mi-proyecto.svc` → forma explícita
- `webapp.mi-proyecto.svc.cluster.local` → FQDN completo

**Registros DNS automáticos:**
- **Services:** `<service>.<ns>.svc.cluster.local` → ClusterIP
- **Headless Services:** Cada Pod obtiene: `<pod-ip>.<service>.<ns>.svc.cluster.local`
- **ExternalName:** Alias a DNS externo

**Ejemplo - Explorar DNS:**

```yaml
# dns-test.yaml
apiVersion: v1
kind: Pod
metadata:
  name: dns-test
spec:
  containers:
  - name: dnsutils
    image: busybox
    command: ["sleep", "3600"]
```

```bash
# Crear pod de prueba
oc apply -f dns-test.yaml

# Probar resolución DNS
oc exec dns-test -- nslookup kubernetes

# Ver desde dentro del Pod
oc exec -it dns-test -- sh
# Dentro del contenedor:
cat /etc/resolv.conf
# search mi-proyecto.svc.cluster.local svc.cluster.local cluster.local
# nameserver 10.96.0.10

nslookup kubernetes
# Server:    10.96.0.10
# Address:   10.96.0.10:53
# Name:      kubernetes.default.svc.cluster.local
# Address:   10.96.0.1

# Probar otros servicios
nslookup webapp
nslookup webapp.otro-proyecto

exit
```

**Configurar DNS personalizado:**

```yaml
# custom-dns.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: openshift-dns
data:
  example.server: |
    example.com:53 {
      forward . 8.8.8.8
    }
```

---

### Routes e Ingress en OpenShift

En Kubernetes se usa Ingress, pero OpenShift usa **Routes**, que son más simples y con más features.

**Tipos de TLS en Routes:**

**Edge (por defecto):**
- El router termina TLS
- El tráfico al backend es HTTP
- Certificado gestionado por OpenShift

**Passthrough:**
- El router NO termina TLS
- El tráfico llega cifrado al Pod
- El Pod debe tener certificado

**Re-encrypt:**
- El router termina TLS del cliente
- Establece nueva conexión TLS al backend
- Útil cuando necesitas TLS end-to-end con certificados diferentes

**Ejemplos de Routes:**

```yaml
# route-edge.yaml - Route con TLS Edge
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: webapp-edge
spec:
  host: webapp.apps.micluster.example.com
  to:
    kind: Service
    name: webapp
  tls:
    termination: edge
    # Certificado generado automáticamente si no se especifica
```

```yaml
# route-passthrough.yaml - Route con TLS Passthrough
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: webapp-passthrough
spec:
  host: secure.apps.micluster.example.com
  to:
    kind: Service
    name: webapp
  tls:
    termination: passthrough
```

```yaml
# route-reencrypt.yaml - Route con TLS Re-encrypt
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: webapp-reencrypt
spec:
  host: secure.apps.micluster.example.com
  to:
    kind: Service
    name: webapp
  tls:
    termination: reencrypt
    certificate: |
      -----BEGIN CERTIFICATE-----
      MIIDXTCCAkWgAwIBAgIJALmVVuBrR...
      -----END CERTIFICATE-----
    key: |
      -----BEGIN RSA PRIVATE KEY-----
      MIIEpAIBAAKCAQEAy7DY6B...
      -----END RSA PRIVATE KEY-----
    caCertificate: |
      -----BEGIN CERTIFICATE-----
      MIIDXTCCAkWgAwIBAgIJALmVV...
      -----END CERTIFICATE-----
    destinationCACertificate: |
      -----BEGIN CERTIFICATE-----
      MIIDXTCCAkWgAwIBAgIJA...
      -----END CERTIFICATE-----
```

**Comandos de gestión:**

```bash
# Crear Route simple
oc expose service webapp

# Crear Route con hostname personalizado
oc expose service webapp --hostname=miapp.apps.cluster.example.com

# Crear Route con TLS
oc create route edge webapp-secure --service=webapp --hostname=secure.apps.cluster.example.com

# Ver Routes
oc get routes
oc get route webapp -o yaml

# Ver certificado de Route
oc get route webapp-secure -o jsonpath='{.spec.tls.certificate}' | openssl x509 -text -noout

# Probar Route
curl -k https://webapp.apps.cluster.example.com
```

**Path-based routing:**

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: multi-path
spec:
  host: api.apps.cluster.example.com
  to:
    kind: Service
    name: api-gateway
    weight: 100
  path: /v1
  wildcardPolicy: None
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: multi-path-v2
spec:
  host: api.apps.cluster.example.com
  to:
    kind: Service
    name: api-gateway-v2
    weight: 100
  path: /v2
  wildcardPolicy: None
```

---

### NetworkPolicies

Por defecto, OpenShift permite todo el tráfico entre Pods. Las NetworkPolicies restringen la comunicación.

**Tipos de reglas:**
- `podSelector` - Selecciona Pods destino
- `namespaceSelector` - Selecciona namespaces
- `ipBlock` - Bloques CIDR de IP
- `ports` - Puertos específicos

**Políticas de entrada/egreso:**
- Sin policy: Todo permitido
- Con policy de ingress: Solo lo especificado permitido
- Con policy de egress: Solo lo especificado permitido

**Ejemplo - Denegar todo el tráfico entrante:**

```yaml
# deny-all-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
spec:
  podSelector: {}  # Aplica a todos los Pods del namespace
  policyTypes:
  - Ingress
  # Sin reglas ingress = todo denegado
```

**Ejemplo - Permitir solo tráfico desde un namespace específico:**

```yaml
# allow-from-namespace.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-frontend
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: frontend
    ports:
    - protocol: TCP
      port: 8080
```

**Ejemplo - Permitir tráfico solo con etiquetas específicas:**

```yaml
# allow-labeled-pods.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-to-api
spec:
  podSelector:
    matchLabels:
      app: api
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: web
    ports:
    - protocol: TCP
      port: 8080
```

**Ejemplo - Egress restringido:**

```yaml
# restrict-egress.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-dns
spec:
  podSelector:
    matchLabels:
      app: webapp
  policyTypes:
  - Egress
  egress:
  # Permitir DNS
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
  # Permitir acceso a servicio específico
  - to:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 8080
```

**Gestión de NetworkPolicies:**

```bash
# Crear NetworkPolicy
oc apply -f deny-all-ingress.yaml

# Ver NetworkPolicies
oc get networkpolicy
oc describe networkpolicy deny-all-ingress

# Probar conectividad
oc exec -it deployment/webapp -- curl backend-service:8080
# Debería fallar si la policy lo deniega

# Eliminar
oc delete networkpolicy deny-all-ingress
```

---

## 2.2 Cargas de trabajo avanzadas

### StatefulSets

StatefulSet gestiona aplicaciones con estado que requieren:
- Identidad de Pod estable (nombre predecible)
- Almacenamiento persistente por Pod
- Orden de despliegue y escalado garantizado

**Diferencias con Deployment:**

| Característica | Deployment | StatefulSet |
|----------------|------------|-------------|
| Nombres de Pods | Aleatorios | Predecibles (app-0, app-1, app-2) |
| Almacenamiento | Compartido | Dedicado por Pod |
| Escalado | Paralelo | Secuencial |
| Actualizaciones | Desordenadas | Ordenadas |
| Network Identity | Ninguna | Headless Service + DNS por Pod |

**Ejemplo - StatefulSet para base de datos:**

```yaml
# statefulset-postgres.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None  # Headless
  selector:
    app: postgres
  ports:
  - port: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          value: "secretpassword"
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

```bash
# Crear StatefulSet
oc apply -f statefulset-postgres.yaml

# Ver StatefulSet y Pods
oc get statefulset
oc get pods -l app=postgres
# NAME         READY   STATUS
# postgres-0   1/1     Running
# postgres-1   1/1     Running
# postgres-2   1/1     Running

# Ver PVCs creados
oc get pvc
# Cada Pod tiene su PVC: data-postgres-0, data-postgres-1, data-postgres-2

# Probar DNS por Pod
oc run test --rm -it --image=busybox -- \
  nslookup postgres-0.postgres-headless

# Escalar (secuencial)
oc scale statefulset postgres --replicas=5
# Crea postgres-3, luego postgres-4

# Reducir (en orden inverso)
oc scale statefulset postgres --replicas=2
# Elimina postgres-4, luego postgres-3
```

---

### DaemonSets

DaemonSet asegura que un Pod corra en **todos** los nodos (o un subset). Usado para:
- Agentes de monitoring (Node Exporter)
- Log collectors (Fluentd)
- Network plugins
- Storage plugins

```yaml
# daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
  labels:
    app: log-collector
spec:
  selector:
    matchLabels:
      app: log-collector
  template:
    metadata:
      labels:
        app: log-collector
    spec:
      # Tolerations para master nodes (opcional)
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
      containers:
      - name: fluentd
        image: fluentd:v1.14
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      terminationGracePeriodSeconds: 30
```

```bash
# Crear DaemonSet
oc apply -f daemonset.yaml

# Ver DaemonSet
oc get daemonset
oc get pods -l app=log-collector -o wide
# Deberías ver un Pod por nodo

# Ver detalles
oc describe daemonset log-collector

# Actualizar DaemonSet
oc set image daemonset/log-collector fluentd=fluentd:v1.15

# Ver rollout
oc rollout status daemonset/log-collector
```

---

### Jobs y CronJobs

**Jobs:** Ejecutan tareas hasta completar con éxito.

**Tipos de Jobs:**
- **Non-parallel:** 1 Pod hasta completar
- **Parallel:** Múltiples Pods en paralelo
- **Work queue:** Los Pods toman trabajo de una cola

```yaml
# job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: backup-job
spec:
  completions: 1           # Número de Pods exitosos necesarios
  parallelism: 1           # Pods en paralelo
  backoffLimit: 4          # Reintentos antes de fallar
  activeDeadlineSeconds: 600  # Timeout total
  template:
    spec:
      restartPolicy: Never  # Job requiere Never o OnFailure
      containers:
      - name: backup
        image: busybox
        command:
        - sh
        - -c
        - |
          echo "Iniciando backup..."
          sleep 10
          echo "Backup completado"
```

**Job paralelo:**

```yaml
# parallel-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: process-items
spec:
  completions: 10       # 10 Pods deben completar
  parallelism: 3        # 3 Pods en paralelo
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: processor
        image: busybox
        command: ["sh", "-c", "echo Procesando item $HOSTNAME && sleep 5"]
```

**CronJob:** Jobs programados con expresión cron.

```yaml
# cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-backup
spec:
  schedule: "0 2 * * *"  # Diario a las 2:00 AM
  concurrencyPolicy: Forbid  # No solapar si el anterior sigue corriendo
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: busybox
            command:
            - sh
            - -c
            - |
              echo "Backup diario - $(date)"
              sleep 30
```

**Gestión de Jobs:**

```bash
# Crear Job
oc apply -f job.yaml

# Ver Jobs
oc get jobs

# Ver Pods del Job
oc get pods -l job-name=backup-job

# Ver logs
oc logs job/backup-job

# Job paralelo
oc apply -f parallel-job.yaml
oc get pods -w  # ver cómo se crean de 3 en 3

# CronJob
oc apply -f cronjob.yaml
oc get cronjobs
oc get jobs  # Jobs creados por el CronJob

# Ejecutar CronJob manualmente
oc create job manual-backup --from=cronjob/daily-backup

# Suspender CronJob
oc patch cronjob daily-backup -p '{"spec":{"suspend":true}}'

# Eliminar Jobs
oc delete job backup-job
oc delete cronjob daily-backup
```

---

### Horizontal Pod Autoscaler (HPA)

HPA escala Deployments/StatefulSets automáticamente basado en métricas:
- CPU
- Memoria
- Métricas custom (Prometheus)

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max
```

**Requisitos:**
- El Deployment debe tener `resources.requests` definidos
- Metrics Server debe estar instalado

```bash
# Verificar metrics server
oc get pods -n openshift-monitoring | grep metrics

# Crear HPA
oc apply -f hpa.yaml

# Ver HPA
oc get hpa
oc describe hpa webapp-hpa

# HPA rápido desde CLI
oc autoscale deployment webapp --min=2 --max=5 --cpu-percent=70

# Probar escalado (generar carga)
oc run load-generator --rm -it --image=busybox -- \
  /bin/sh -c "while sleep 0.01; do wget -q -O- http://webapp; done"

# Ver escalado en tiempo real
oc get hpa webapp-hpa -w

# Ver historial de escalado
oc describe hpa webapp-hpa | grep -A10 Events
```

---

## 2.3 Seguridad en el clúster

### RBAC: Roles, ClusterRoles y RoleBindings

**RBAC** (Role-Based Access Control) define quién puede hacer qué en el clúster.

**Componentes:**
- **Role:** Permisos dentro de un namespace
- **ClusterRole:** Permisos a nivel de clúster
- **RoleBinding:** Vincula Role a usuarios/grupos en un namespace
- **ClusterRoleBinding:** Vincula ClusterRole a usuarios/grupos a nivel global

```yaml
# role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-role
  namespace: mi-proyecto
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "secrets"]
  verbs: ["get", "list", "watch", "create", "update", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "delete"]
- apiGroups: ["route.openshift.io"]
  resources: ["routes"]
  verbs: ["get", "list", "watch", "create", "update", "delete"]
```

```yaml
# rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: mi-proyecto
subjects:
- kind: User
  name: developer1
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
```

**ClusterRole de solo lectura:**

```yaml
# clusterrole-view.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-viewer
rules:
- apiGroups: [""]
  resources: ["pods", "services", "nodes", "namespaces"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "daemonsets", "statefulsets"]
  verbs: ["get", "list", "watch"]
```

**Roles predefinidos en OpenShift:**
- `admin` - Control total del proyecto
- `edit` - Crear/modificar recursos, sin roles ni security
- `view` - Solo lectura
- `cluster-admin` - Super-admin del clúster

```bash
# Ver roles en namespace
oc get roles

# Ver ClusterRoles
oc get clusterroles | head -20

# Ver RoleBindings
oc get rolebindings

# Asignar rol predefinido
oc adm policy add-role-to-user edit developer1 -n mi-proyecto
oc adm policy add-role-to-user admin admin1 -n mi-proyecto

# Asignar rol de cluster-admin (con cuidado)
oc adm policy add-cluster-role-to-user cluster-admin admin

# Ver permisos de usuario
oc auth can-i --list --as=developer1
oc auth can-i create pods --as=developer1 -n mi-proyecto
oc auth can-i delete nodes --as=developer1

# Eliminar binding
oc adm policy remove-role-from-user edit developer1 -n mi-proyecto
```

---

### ServiceAccounts

ServiceAccounts representan identidades para procesos dentro del clúster.

**ServiceAccount por defecto:**
Cada namespace tiene `default` SA, usada por Pods sin SA especificado.

```yaml
# serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-bot
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: builder-role
rules:
- apiGroups: [""]
  resources: ["pods", "configmaps"]
  verbs: ["get", "list", "watch", "create", "delete"]
- apiGroups: ["image.openshift.io"]
  resources: ["imagestreams"]
  verbs: ["get", "list", "watch", "update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: build-bot-binding
subjects:
- kind: ServiceAccount
  name: build-bot
roleRef:
  kind: Role
  name: builder-role
```

**Usar ServiceAccount en Pod:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api-pod
spec:
  serviceAccountName: build-bot
  containers:
  - name: api
    image: busybox
    command: ["sleep", "3600"]
```

```bash
# Crear ServiceAccount
oc create serviceaccount build-bot

# Ver ServiceAccounts
oc get serviceaccounts

# Ver token de ServiceAccount (montado en Pods)
oc describe serviceaccount build-bot

# Asignar rol
oc adm policy add-role-to-user edit -z build-bot

# Ver secrets del SA
oc get secrets | grep build-bot

# Token JWT para APIs externas
oc create token build-bot --duration=24h
```

---

### Pod Security Admission (PSA)

PSA reemplaza PodSecurityPolicies (deprecados). Controla permisos de seguridad de Pods.

**Niveles de PSA:**
- **Privileged:** Sin restricciones
- **Baseline:** Restricciones mínimas
- **Restricted:** Restricciones fuertes (requiere runAsNonRoot, etc.)

**Modos:**
- **Enforce:** Bloquea violaciones
- **Audit:** Registra pero permite
- **Warn:** Avisa pero permite

**Configurar PSA en namespace:**

```yaml
# namespace-psa.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: secure-apps
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

**Pod compatible con restricted:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx:alpine
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
```

```bash
# Verificar PSA en namespace
oc get namespace secure-apps -o yaml | grep pod-security

# Probar Pod que viola PSA
cat <<EOF | oc apply -f - --dry-run=client
apiVersion: v1
kind: Pod
metadata:
  name: insecure-pod
  namespace: secure-apps
spec:
  containers:
  - name: app
    image: nginx
EOF
# Warning: would violate PodSecurity

# Verificar PSA activo
oc auth can-i create pods --as=system:anonymous -n secure-apps
```

---

### Gestión de secretos: buenas prácticas

**Problema:** Secrets en base64 NO están encriptados, solo codificados.

**Mejores prácticas:**

**1. No versionar Secrets en Git:**
```bash
# Mal: commit del secret
git add secret.yaml  # NO

# Bien: usar SealedSecrets, External Secrets Operator, o inyección en CI/CD
```

**2. Cifrado en reposo:**
```bash
# Verificar cifrado activo
oc get encryptionconfig -o yaml
```

**3. RBAC estricto:**
```bash
# Solo ServiceAccounts necesarios pueden leer secrets
oc auth can-i get secrets --as=system:serviceaccount:mi-proyecto:default
```

**4. Usar proveedores externos:**
- HashiCorp Vault
- AWS Secrets Manager
- Azure Key Vault
- External Secrets Operator

**Ejemplo con External Secrets Operator:**

```yaml
# ExternalSecret apunta a Vault
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: db-credentials-synced
    creationPolicy: Owner
  data:
  - secretKey: password
    remoteRef:
      key: secret/data/database
      property: password
```

**5. Rotación de secrets:**

```yaml
# Job que rota secret periódicamente
apiVersion: batch/v1
kind: CronJob
metadata:
  name: rotate-db-password
spec:
  schedule: "0 0 1 * *"  # Mensual
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: secret-rotator
          containers:
          - name: rotator
            image: bitnami/kubectl
            command:
            - /bin/sh
            - -c
            - |
              NEW_PASS=$(openssl rand -base64 24)
              kubectl patch secret db-credentials \
                -p="{\"stringData\":{\"password\":\"$NEW_PASS\"}}"
              kubectl rollout restart deployment/webapp
          restartPolicy: OnFailure
```

---

## 2.4 Gestión de recursos

### Requests y Limits de CPU y memoria

**Requests:** Lo que el Pod garantiza recibir (usado para scheduling).
**Limits:** Máximo que el Pod puede consumir.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: 100m       # 0.1 cores (100 millicores)
        memory: 128Mi   # 128 mebibytes
      limits:
        cpu: 500m       # 0.5 cores máximo
        memory: 256Mi   # 256 MiB máximo
```

**Unidades:**
- CPU: `m` (millicores), `1` = 1 core, `100m` = 0.1 cores
- Memoria: `Ki`, `Mi`, `Gi`, `Ti` (potencias de 1024), `K`, `M`, `G` (potencias de 1000)

**Comportamiento:**
- CPU: Se limita con throttling (el proceso se ralentiza)
- Memoria: Si excede limit → OOMKilled (terminado)

```bash
# Ver recursos de Pods
oc get pods -o custom-columns=NAME:.metadata.name,\
CPU_REQ:.spec.containers[*].resources.requests.cpu,\
MEM_REQ:.spec.containers[*].resources.requests.memory,\
CPU_LIM:.spec.containers[*].resources.limits.cpu,\
MEM_LIM:.spec.containers[*].resources.limits.memory

# Ver uso real
oc adm top pods
oc adm top nodes

# Stress test para ver limits
oc run stress --rm -it --image=progrium/stress -- \
  --cpu 2 --timeout 30s
# Si el limit es 500m, se throttleará
```

---

### LimitRange y ResourceQuota

**LimitRange:** Límites por defecto y máximos a nivel de container/pod.

```yaml
# limitrange.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: project-limits
spec:
  limits:
  # Límites para containers
  - type: Container
    default:          # Limits por defecto
      cpu: 500m
      memory: 512Mi
    defaultRequest:   # Requests por defecto
      cpu: 100m
      memory: 128Mi
    max:              # Máximo permitido
      cpu: 2
      memory: 2Gi
    min:              # Mínimo permitido
      cpu: 10m
      memory: 16Mi
  # Límites para PVCs
  - type: PersistentVolumeClaim
    max:
      storage: 10Gi
    min:
      storage: 100Mi
```

**ResourceQuota:** Límites agregados a nivel de namespace.

```yaml
# resourcequota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: project-quota
spec:
  hard:
    # Conteo de objetos
    pods: 10
    services: 5
    secrets: 10
    configmaps: 10
    persistentvolumeclaims: 5
    
    # Recursos compute
    requests.cpu: 4
    requests.memory: 8Gi
    limits.cpu: 8
    limits.memory: 16Gi
    
    # Almacenamiento
    requests.storage: 50Gi
```

```bash
# Aplicar LimitRange y ResourceQuota
oc apply -f limitrange.yaml
oc apply -f resourcequota.yaml

# Ver LimitRange
oc get limitrange
oc describe limitrange project-limits

# Ver ResourceQuota
oc get resourcequota
oc describe resourcequota project-quota

# Ver uso de quota
oc describe quota

# Probar límite excedido
oc run too-many-pods --image=nginx --replicas=15
# Error: exceeded quota: project-quota, requested: pods=15
```

---

### Quality of Service: Guaranteed, Burstable, BestEffort

QoS se asigna automáticamente según requests/limits:

**Guaranteed:**
- Todos los containers tienen requests y limits
- limits == requests para CPU y memoria
- Último en ser evictado

**Burstable:**
- Al menos un container tiene requests o limits
- No cumple Guaranteed
- Segundo en ser evictado

**BestEffort:**
- Ningún container tiene requests ni limits
- Primero en ser evictado bajo presión

```yaml
# Guaranteed QoS
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: 500m
        memory: 256Mi
      limits:
        cpu: 500m      # == request
        memory: 256Mi  # == request
```

```yaml
# Burstable QoS
apiVersion: v1
kind: Pod
metadata:
  name: burstable
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 500m      # != request
        memory: 512Mi  # != request
```

```yaml
# BestEffort QoS
apiVersion: v1
kind: Pod
metadata:
  name: besteffort
spec:
  containers:
  - name: app
    image: nginx
    # Sin resources
```

```bash
# Ver QoS de Pods
oc get pods -o custom-columns=NAME:.metadata.name,\
QOS:.status.qosClass

# O con describe
oc describe pod guaranteed | grep "QoS Class"
```

---

## Ejercicio práctico integrador - Día 2

Despliega una arquitectura completa con networking, seguridad y gestión de recursos:

```yaml
# ejercicio-dia2.yaml
# Namespace con PSA
---
apiVersion: v1
kind: Namespace
metadata:
  name: ejercicio-dia2
  labels:
    pod-security.kubernetes.io/enforce: baseline
---
# ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: ejercicio-dia2
---
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: ejercicio-dia2
data:
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"
---
# Secret
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: ejercicio-dia2
stringData:
  API_KEY: "super-secret-key-12345"
---
# NetworkPolicy - Solo permitir tráfico interno
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-internal-only
  namespace: ejercicio-dia2
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
  egress:
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
---
# Backend StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: backend
  namespace: ejercicio-dia2
spec:
  serviceName: backend-headless
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      serviceAccountName: app-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
      containers:
      - name: app
        image: nginx:alpine
        ports:
        - containerPort: 80
        envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: app-secret
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 300m
            memory: 256Mi
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 100Mi
---
# Headless Service para StatefulSet
apiVersion: v1
kind: Service
metadata:
  name: backend-headless
  namespace: ejercicio-dia2
spec:
  clusterIP: None
  selector:
    app: backend
  ports:
  - port: 80
---
# Service principal para backend
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: ejercicio-dia2
spec:
  selector:
    app: backend
  ports:
  - port: 80
---
# Frontend Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: ejercicio-dia2
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      serviceAccountName: app-sa
      containers:
      - name: web
        image: nginx:alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 100m
            memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: ejercicio-dia2
spec:
  selector:
    app: frontend
  ports:
  - port: 80
---
# Route
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: frontend
  namespace: ejercicio-dia2
spec:
  to:
    kind: Service
    name: frontend
  tls:
    termination: edge
---
# HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-hpa
  namespace: ejercicio-dia2
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: frontend
  minReplicas: 2
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
---
# ResourceQuota
apiVersion: v1
kind: ResourceQuota
metadata:
  name: project-quota
  namespace: ejercicio-dia2
spec:
  hard:
    pods: 15
    requests.cpu: 2
    requests.memory: 2Gi
    limits.cpu: 4
    limits.memory: 4Gi
```

```bash
# Desplegar todo
oc apply -f ejercicio-dia2.yaml

# Verificar
oc get all -n ejercicio-dia2
oc get networkpolicy,resourcequota,hpa -n ejercicio-dia2

# Ver StatefulSet con PVCs
oc get pvc -n ejercicio-dia2

# Ver QoS
oc get pods -n ejercicio-dia2 -o custom-columns=NAME:.metadata.name,QOS:.status.qosClass

# Probar Route
oc get route frontend -n ejercicio-dia2
curl -k https://$(oc get route frontend -n ejercicio-dia2 -o jsonpath='{.spec.host}')

# Limpiar
oc delete namespace ejercicio-dia2
```

---

## Resumen del Día 2

**Conceptos clave:**
- OVN-Kubernetes = CNI con overlay, network policies, IPsec
- DNS interno = CoreDNS con nombres `<svc>.<ns>.svc.cluster.local`
- Routes = Exposición HTTP/HTTPS con TLS (Edge, Passthrough, Re-encrypt)
- NetworkPolicies = Aislamiento de tráfico entre Pods
- StatefulSet = Apps con estado (nombres estables, PVCs dedicados)
- DaemonSet = Un Pod por nodo (logging, monitoring)
- Jobs/CronJobs = Tareas batch y programadas
- HPA = Escalado automático basado en métricas
- RBAC = Control de acceso con Roles y RoleBindings
- ServiceAccounts = Identidad para Pods
- PSA = Restricciones de seguridad (Privileged, Baseline, Restricted)
- Requests/Limits = Garantías y límites de recursos
- LimitRange = Límites por defecto/máximos por container
- ResourceQuota = Cuotas agregadas por namespace
- QoS = Guaranteed > Burstable > BestEffort
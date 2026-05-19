# OpenShift Container Platform - Día 4
## OpenShift: Plataforma Avanzada

---

## 4.1 Scheduling avanzado

### Node selectors, taints y tolerations

**Node Selectors:** Filtro simple para dirigir Pods a nodos específicos.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  nodeSelector:
    hardware: gpu
  containers:
  - name: ml-training
    image: tensorflow/tensorflow:latest-gpu
```

```bash
# Etiquetar nodos
oc label node worker-0 hardware=gpu
oc label node worker-1 hardware=cpu
oc label node worker-2 environment=production

# Ver labels
oc get nodes --show-labels
oc get nodes -L hardware,environment

# Eliminar label
oc label node worker-0 hardware-
```

**Taints y Tolerations:** Control más granular para repeler o atraer Pods.

**Taints en nodos:** "Manchas" que repelen Pods sin toleration correspondiente.
- Effect: `NoSchedule` (no programa), `PreferNoSchedule` (evita si posible), `NoExecute` (evicta Pods existentes)

```bash
# Aplicar taint
oc adm taint nodes worker-0 hardware=gpu:NoSchedule
oc adm taint nodes worker-1 dedicated=infra:NoExecute
oc adm taint nodes worker-2 special=true:PreferNoSchedule

# Ver taints
oc describe node worker-0 | grep Taints

# Eliminar taint
oc adm taint nodes worker-0 hardware:NoSchedule-
```

**Tolerations en Pods:** Permite que un Pod se programe en nodos con taints específicos.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-workload
spec:
  tolerations:
  - key: "hardware"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
  - key: "dedicated"
    operator: "Equal"
    value: "infra"
    effect: "NoExecute"
  # Toleración wildcard
  - key: "special"
    operator: "Exists"
    effect: "NoSchedule"
  containers:
  - name: app
    image: nvidia/cuda:latest
```

**Ejemplo completo - Nodos dedicados:**

```bash
# Configurar nodos dedicados a infraestructura
oc label node worker-0 node-role.kubernetes.io/infra=""
oc adm taint nodes worker-0 node-role.kubernetes.io/infra=:NoSchedule

# Mover componentes de OpenShift a nodos infra
oc patch ingresscontroller/default -n openshift-ingress-operator \
  --type=merge -p '{"spec":{"nodePlacement":{"nodeSelector":{"matchLabels":{"node-role.kubernetes.io/infra":""}},"tolerations":[{"key":"node-role.kubernetes.io/infra","operator":"Exists","effect":"NoSchedule"}]}}}'

oc patch config cluster -n openshift-image-registry \
  --type=merge -p '{"spec":{"affinity":{"nodeAffinity":{"requiredDuringSchedulingIgnoredDuringExecution":{"nodeSelectorTerms":[{"matchExpressions":[{"key":"node-role.kubernetes.io/infra","operator":"Exists"]}]}}}}}'
```

---

### Pod Affinity y Anti-Affinity

**Affinity:** Preferencia para programar Pods cerca (o lejos) de otros Pods.

**Tipos:**
- `requiredDuringSchedulingIgnoredDuringExecution` - Hard requirement
- `preferredDuringSchedulingIgnoredDuringExecution` - Soft preference

**Pod Affinity (co-locación):**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      affinity:
        podAffinity:
          # Requerido: debe estar en el mismo nodo que el backend
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - backend
            topologyKey: kubernetes.io/hostname
      containers:
      - name: frontend
        image: nginx
```

**Pod Anti-Affinity (dispersión):**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      affinity:
        podAntiAffinity:
          # Preferido: evitar estar en el mismo nodo que otros backends
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - backend
              topologyKey: kubernetes.io/hostname
          # Requerido: evitar estar en el mismo AZ que otros backends
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - backend
            topologyKey: topology.kubernetes.io/zone
      containers:
      - name: backend
        image: node:18-alpine
```

**Node Affinity (programación por labels de nodo):**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: high-memory-app
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: memory
            operator: Gt
            values:
            - "64"  # Más de 64GB RAM
          - key: environment
            operator: In
            values:
            - production
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values:
            - us-east-1a
  containers:
  - name: app
    image: myapp:latest
```

**Ejemplo completo - Alta disponibilidad:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ha-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ha-app
  template:
    metadata:
      labels:
        app: ha-app
    spec:
      affinity:
        # Dispersar pods en diferentes nodos
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: ha-app
            topologyKey: kubernetes.io/hostname
        # Dispersar pods en diferentes zonas
        preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app: ha-app
            topologyKey: topology.kubernetes.io/zone
        # Preferir nodos con SSD
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 50
            preference:
              matchExpressions:
              - key: disk
                operator: In
                values:
                - ssd
      containers:
      - name: app
        image: nginx
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
```

---

### Priority Classes

**PriorityClass:** Define la prioridad de los Pods. Pods de alta prioridad pueden desalojar Pods de baja prioridad (preemption).

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "Usar para pods críticos"
preemptionPolicy: PreemptLowerPriority  # Por defecto
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: medium-priority
value: 100000
globalDefault: true  # Default para pods sin priorityClass
description: "Prioridad media"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 1000
description: "Prioridad baja"
preemptionPolicy: Never  # No desaloja otros pods
```

**Usar PriorityClass en Pod:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: critical-pod
spec:
  priorityClassName: high-priority
  containers:
  - name: app
    image: nginx
```

```bash
# Ver PriorityClasses
oc get priorityclass

# Ver priority de un Pod
oc get pod critical-pod -o jsonpath='{.spec.priorityClassName}'

# PriorityClasses predefinidas en OpenShift
oc get priorityclass
# system-cluster-critical
# system-node-critical
# openshift-user-critical
```

---

## 4.2 Helm en OpenShift

### Estructura de un chart

**Helm** es un gestor de paquetes para Kubernetes. Un chart es un paquete Helm.

**Estructura de directorios:**

```
mychart/
├── Chart.yaml          # Metadata del chart
├── values.yaml         # Valores por defecto
├── charts/             # Dependencias
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── _helpers.tpl    # Plantillas reutilizables
│   └── NOTES.txt       # Notas post-instalación
├── templates/tests/    # Tests del chart
│   └── test-connection.yaml
└── .helmignore         # Archivos a ignorar
```

**Chart.yaml:**

```yaml
apiVersion: v2
name: myapp
description: A Helm chart for my application
type: application
version: 1.0.0         # Versión del chart
appVersion: "2.0.0"    # Versión de la aplicación
kubeVersion: ">=1.20.0-0"
home: https://myapp.example.com
sources:
  - https://github.com/myorg/myapp
maintainers:
  - name: myteam
    email: team@example.com
dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
```

**values.yaml:**

```yaml
# Valores por defecto
replicaCount: 2

image:
  repository: myapp
  tag: latest
  pullPolicy: IfNotPresent

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  name: ""

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts:
    - host: myapp.local
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

nodeSelector: {}
tolerations: []
affinity: {}
```

**templates/deployment.yaml:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "myapp.serviceAccountName" . }}
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.port }}
              protocol: TCP
          {{- if .Values.resources }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          {{- end }}
          {{- with .Values.env }}
          env:
            {{- toYaml . | nindent 12 }}
          {{- end }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

**templates/_helpers.tpl:**

```yaml
{{/* vim: set filetype=mustache: */}}
{{/*
Expand the name of the chart.
*/}}
{{- define "myapp.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "myapp.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Create chart name and version as used by the chart label.
*/}}
{{- define "myapp.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "myapp.labels" -}}
helm.sh/chart: {{ include "myapp.chart" . }}
{{ include "myapp.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "myapp.selectorLabels" -}}
app.kubernetes.io/name: {{ include "myapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Create the name of the service account to use
*/}}
{{- define "myapp.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "myapp.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

---

### Instalación, actualización y rollback de releases

**Instalación:**

```bash
# Instalar repositorio de Helm
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Buscar charts
helm search repo nginx
helm search hub wordpress  # Artifact Hub

# Instalar chart
helm install my-release bitnami/nginx

# Instalar con valores custom
helm install my-release bitnami/nginx -f values.yaml

# Instalar con valores inline
helm install my-release bitnami/nginx --set replicaCount=3

# Dry-run (verificar sin instalar)
helm install my-release bitnami/nginx --dry-run --debug

# Instalar en namespace específico
helm install my-release bitnami/nginx -n my-namespace --create-namespace
```

**Actualización:**

```bash
# Ver releases
helm list
helm list -A  # todos los namespaces

# Actualizar valores
helm upgrade my-release bitnami/nginx --set replicaCount=5

# Actualizar con archivo de valores
helm upgrade my-release bitnami/nginx -f values.yaml

# Upgrade con rollback automático si falla
helm upgrade my-release bitnami/nginx --atomic

# Ver historial
helm history my-release

# Ver manifiestos de una revisión
helm get manifest my-release --revision 2

# Ver valores de una revisión
helm get values my-release --revision 2
```

**Rollback:**

```bash
# Rollback a revisión anterior
helm rollback my-release 1

# Ver estado después del rollback
helm status my-release
```

**Desinstalación:**

```bash
# Desinstalar release
helm uninstall my-release

# Conservar historial
helm uninstall my-release --keep-history
```

---

### Repositorios de charts y customización con values.yaml

**Añadir repositorios:**

```bash
# Repositorios populares
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

# Actualizar repos
helm repo update

# Listar repos
helm repo list
```

**Customización avanzada con values.yaml:**

```yaml
# values-production.yaml
replicaCount: 5

image:
  repository: quay.io/myorg/myapp
  tag: "v2.0.0"
  pullPolicy: Always

imagePullSecrets:
  - name: quay-secret

service:
  type: LoadBalancer
  port: 443
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: arn:aws:acm:...

ingress:
  enabled: true
  className: nginx
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: myapp-tls
      hosts:
        - myapp.example.com

resources:
  limits:
    cpu: 2000m
    memory: 2Gi
  requests:
    cpu: 1000m
    memory: 1Gi

autoscaling:
  enabled: true
  minReplicas: 5
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

nodeSelector:
  environment: production

tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "production"
    effect: "NoSchedule"

affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
          - key: app.kubernetes.io/name
            operator: In
            values:
            - myapp
        topologyKey: topology.kubernetes.io/zone

# Dependencias
postgresql:
  enabled: true
  auth:
    postgresPassword: supersecret
    database: myapp
  primary:
    persistence:
      size: 100Gi
```

```bash
# Instalar con values de producción
helm install myapp ./mychart -f values-production.yaml

# Múltiples archivos de valores (se mergean)
helm install myapp ./mychart \
  -f values.yaml \
  -f values-production.yaml \
  -f values-secrets.yaml

# Ver valores efectivos
helm get values myapp --all
```

**Desarrollo local de charts:**

```bash
# Crear chart desde template
helm create mychart

# Validar chart
helm lint mychart

# Empaquetar chart
helm package mychart

# Instalar desde tarball
helm install myapp mychart-1.0.0.tgz

# Serv repositorio local
helm serve &
helm repo add local http://localhost:8879
```

---

## 4.3 Operators y ciclo de vida de la plataforma

### Qué es un Operator y el patrón de reconciliación

Un **Operator** es un método de empaquetar, desplegar y gestionar una aplicación Kubernetes usando APIs declarativas.

**Patrón de reconciliación:**
```
┌────────────────────────────────────────────┐
│              Control Loop                   │
│  ┌──────────┐        ┌──────────────────┐  │
│  │  Watch   │───────►│ Desired State    │  │
│  │  Events  │        │ (Custom Resource)│  │
│  └──────────┘        └────────┬─────────┘  │
│                               │             │
│                               ▼             │
│                      ┌─────────────────┐   │
│                      │ Compare with    │   │
│                      │ Current State   │   │
│                      └────────┬────────┘   │
│                               │             │
│                    ┌──────────┴─────────┐   │
│                    │                    │   │
│                    ▼                    ▼   │
│              ┌──────────┐        ┌─────────┐│
│              │  Reconcile│        │  Do     ││
│              │  (Update) │        │  Nothing││
│              └──────────┘        └─────────┘│
└────────────────────────────────────────────┘
```

**Componentes de un Operator:**
- **CRD (Custom Resource Definition):** Define el nuevo tipo de recurso
- **CR (Custom Resource):** Instancia del recurso
- **Controller:** Implementa el loop de reconciliación

**Ejemplo - CRD simple:**

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.myapp.example.com
spec:
  group: myapp.example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                version:
                  type: string
                storageSize:
                  type: string
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 10
            status:
              type: object
              properties:
                conditions:
                  type: array
                  items:
                    type: object
                    properties:
                      type:
                        type: string
                      status:
                        type: string
  scope: Namespaced
  names:
    plural: databases
    singular: database
    kind: Database
    shortNames:
    - db
```

**CR ejemplo:**

```yaml
apiVersion: myapp.example.com/v1
kind: Database
metadata:
  name: my-database
spec:
  version: "15"
  storageSize: 10Gi
  replicas: 3
```

---

### OperatorHub y Operator Lifecycle Manager (OLM)

**OperatorHub:** Catálogo de Operators certificados y comunitarios.

**OLM (Operator Lifecycle Manager):** Gestiona el ciclo de vida de Operators (instalación, actualización, permisos).

**Instalar Operator desde consola:**
1. Navegar a Operators → OperatorHub
2. Buscar el Operator deseado
3. Click Install
4. Seleccionar namespace y modo de instalación
5. Click Subscribe

**Instalar Operator desde CLI:**

```bash
# Ver Operators disponibles
oc get packagemanifests -n openshift-marketplace

# Buscar Operator
oc search operator postgresql

# Ver detalles
oc describe packagemanifest postgresql-operator -n openshift-marketplace

# Crear Subscription (instala el Operator)
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: postgresql-operator
  namespace: openshift-operators
spec:
  channel: stable
  name: postgresql-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
EOF

# Ver estado de la instalación
oc get clusterserviceversion -n openshift-operators
oc get installplan -n openshift-operators

# Ver Operator desplegado
oc get pods -n openshift-operators -l name=postgresql-operator
```

**CatalogSources:** Fuentes de Operators.

```bash
# Ver catálogos disponibles
oc get catalogsources -n openshift-marketplace

# Catálogos por defecto
# redhat-operators - Operators certificados por Red Hat
# certified-operators - Partners certificados
# community-operators - Operators comunitarios
# redhat-marketplace - Marketplace de Red Hat
```

**Crear Instance de un Operator:**

```yaml
# Una vez instalado el Operator, crear una instancia
apiVersion: postgresql.k8s.enterprisedb.io/v1
kind: Cluster
metadata:
  name: postgresql-cluster
spec:
  instances: 3
  primaryUpdateStrategy: unsupervised
  
  storage:
    size: 1Gi
  
  bootstrap:
    initdb:
      database: myapp
      owner: myappuser
  
  resources:
    requests:
      memory: "512Mi"
      cpu: "1"
    limits:
      memory: "1Gi"
      cpu: "2"
```

---

### Operators de plataforma: Pipelines, GitOps, Logging, Service Mesh

**OpenShift Pipelines (Tekton):**

```bash
# Instalar OpenShift Pipelines Operator
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-pipelines-operator
  namespace: openshift-operators
spec:
  channel: stable
  name: openshift-pipelines-operator-rh
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

# Verificar instalación
oc get pods -n openshift-pipelines
```

**Pipeline Tekton ejemplo:**

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: build-and-deploy
spec:
  workspaces:
  - name: shared-workspace
  params:
  - name: git-url
    type: string
  - name: image-name
    type: string
  tasks:
  - name: fetch-repository
    taskRef:
      name: git-clone
    workspaces:
    - name: output
      workspace: shared-workspace
    params:
    - name: url
      value: $(params.git-url)
  
  - name: build-image
    taskRef:
      name: buildah
    runAfter:
    - fetch-repository
    workspaces:
    - name: source
      workspace: shared-workspace
    params:
    - name: IMAGE
      value: $(params.image-name)
  
  - name: deploy
    taskRef:
      name: openshift-client
    runAfter:
    - build-image
    params:
    - name: SCRIPT
      value: |
        oc apply -f $(workspaces.source.path)/k8s/deployment.yaml
```

**OpenShift GitOps (ArgoCD):**

```bash
# Instalar OpenShift GitOps Operator
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-gitops-operator
  namespace: openshift-operators
spec:
  channel: stable
  name: openshift-gitops-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

# Acceder a ArgoCD
oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}'

# Password inicial
oc get secret openshift-gitops-cluster -n openshift-gitops -o jsonpath='{.data.admin\.password}' | base64 -d
```

**Application ArgoCD ejemplo:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: openshift-gitops
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/myapp-gitops.git
    targetRevision: HEAD
    path: overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp-production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

**OpenShift Logging:**

```bash
# Instalar OpenShift Logging Operator
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: cluster-logging
  namespace: openshift-logging
spec:
  channel: stable
  name: cluster-logging
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

# Crear ClusterLogging instance
oc apply -f - <<EOF
apiVersion: logging.openshift.io/v1
kind: ClusterLogging
metadata:
  name: instance
  namespace: openshift-logging
spec:
  managementState: Managed
  logStore:
    type: elasticsearch
    elasticsearch:
      nodeCount: 3
      resources:
        requests:
          memory: 8Gi
      storage:
        storageClassName: gp2
        size: 100Gi
  visualization:
    type: kibana
    kibana:
      replicas: 1
  collection:
    logs:
      type: fluentd
      fluentd: {}
EOF
```

**OpenShift Service Mesh:**

```bash
# Instalar Service Mesh Operators
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: servicemeshoperator
  namespace: openshift-operators
spec:
  channel: stable
  name: servicemeshoperator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: jaeger-product
  namespace: openshift-operators
spec:
  channel: stable
  name: jaeger-product
  source: redhat-operators
  sourceNamespace: openshift-marketplace
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: kiali-ossm
  namespace: openshift-operators
spec:
  channel: stable
  name: kiali-ossm
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

---

### Cluster Version Operator (CVO) y actualizaciones del clúster

**CVO** gestiona las actualizaciones de OpenShift.

```bash
# Ver versión actual
oc get clusterversion
oc describe clusterversion

# Ver canales disponibles
oc adm upgrade

# Ver historial de actualizaciones
oc get clusterversion version -o jsonpath='{.status.history}'

# Actualizar clúster
oc adm upgrade --to=4.14.0

# Actualizar al último del canal
oc adm upgrade --to-latest

# Ver progreso
oc get clusterversion -w

# Ver operators pendientes de actualizar
oc get clusteroperators | grep -v "True\sTrue\sTrue"
```

**Configurar canal de actualizaciones:**

```yaml
apiVersion: config.openshift.io/v1
kind: ClusterVersion
metadata:
  name: version
spec:
  channel: stable-4.14
  upstream: https://api.openshift.com/api/upgrades_info/v1/graph
```

---

### Machine Config Operator (MCO) y MachineConfigPools

**MCO** gestiona la configuración de los nodos del clúster.

**MachineConfigPools:** Grupos de máquinas con la misma configuración.

```bash
# Ver MachineConfigPools
oc get machineconfigpools
oc describe mcp master
oc describe mcp worker

# Ver MachineConfigs
oc get machineconfigs

# Ver configuración aplicada a un nodo
oc get node worker-0 -o jsonpath='{.metadata.annotations.machineconfiguration\.openshift\.io/currentConfig}'
```

**Crear MachineConfig custom:**

```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker
  name: custom-worker-config
spec:
  config:
    ignition:
      version: 3.2.0
    storage:
      files:
      - contents:
          source: data:text/plain;charset=utf-8;base64,IyBUZXN0IGZpbGUK
        mode: 0644
        path: /etc/custom/config.conf
        overwrite: true
    systemd:
      units:
      - name: custom-service.service
        enabled: true
        contents: |
          [Unit]
          Description=Custom Service
          After=network.target

          [Service]
          Type=simple
          ExecStart=/usr/local/bin/custom-script.sh
          Restart=always

          [Install]
          WantedBy=multi-user.target
```

```bash
# Aplicar MachineConfig
oc apply -f machineconfig.yaml

# Ver progreso de aplicación
oc get mcp worker -w

# Ver nodos actualizados
oc get nodes -l node-role.kubernetes.io/worker
```

---

## 4.4 CI/CD y GitOps

### OpenShift Pipelines (Tekton): Tasks, Pipelines, Triggers

Ya cubierto parcialmente arriba. Aquí ejemplos adicionales:

**Task reutilizable:**

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: npm-build
spec:
  workspaces:
  - name: source
  params:
  - name: contextDir
    default: "."
  steps:
  - name: install
    image: node:18-alpine
    workingDir: $(workspaces.source.path)/$(params.contextDir)
    script: |
      npm ci
  - name: test
    image: node:18-alpine
    workingDir: $(workspaces.source.path)/$(params.contextDir)
    script: |
      npm test
  - name: build
    image: node:18-alpine
    workingDir: $(workspaces.source.path)/$(params.contextDir)
    script: |
      npm run build
```

**Pipeline completo CI/CD:**

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: ci-pipeline
spec:
  workspaces:
  - name: shared-workspace
  params:
  - name: git-url
  - name: git-revision
    default: main
  - name: image-registry
    default: image-registry.openshift-image-registry.svc:5000
  - name: image-name
  tasks:
  - name: clone
    taskRef:
      name: git-clone
    workspaces:
    - name: output
      workspace: shared-workspace
    params:
    - name: url
      value: $(params.git-url)
    - name: revision
      value: $(params.git-revision)
  
  - name: build-and-test
    runAfter: [clone]
    taskRef:
      name: npm-build
    workspaces:
    - name: source
      workspace: shared-workspace
  
  - name: build-image
    runAfter: [build-and-test]
    taskRef:
      name: buildah
    workspaces:
    - name: source
      workspace: shared-workspace
    params:
    - name: IMAGE
      value: $(params.image-registry)/$(params.image-name):$(params.git-revision)
  
  - name: deploy-dev
    runAfter: [build-image]
    taskRef:
      name: openshift-client
    params:
    - name: SCRIPT
      value: |
        oc set image deployment/myapp myapp=$(params.image-registry)/$(params.image-name):$(params.git-revision) -n dev
```

**Trigger para webhooks:**

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata:
  name: ci-trigger-template
spec:
  params:
  - name: git-repo-url
  - name: git-revision
  resourcetemplates:
  - apiVersion: tekton.dev/v1beta1
    kind: PipelineRun
    metadata:
      generateName: ci-run-
    spec:
      pipelineRef:
        name: ci-pipeline
      workspaces:
      - name: shared-workspace
        volumeClaimTemplate:
          spec:
            accessModes:
            - ReadWriteOnce
            resources:
              requests:
                storage: 1Gi
      params:
      - name: git-url
        value: $(tt.params.git-repo-url)
      - name: git-revision
        value: $(tt.params.git-revision)
---
apiVersion: triggers.tekton.dev/v1beta1
kind: EventListener
metadata:
  name: ci-eventlistener
spec:
  serviceAccountName: pipeline
  triggers:
  - name: github-trigger
    bindings:
    - name: git-repo-url
      value: $(body.repository.clone_url)
    - name: git-revision
      value: $(body.after)
    template:
      name: ci-trigger-template
```

---

### OpenShift GitOps (ArgoCD): Applications, sincronización, entornos

**Configuración de entornos con Kustomize:**

Estructura de repositorio GitOps:
```
gitops-repo/
├── apps/
│   └── myapp/
│       ├── base/
│       │   ├── deployment.yaml
│       │   ├── service.yaml
│       │   ├── configmap.yaml
│       │   └── kustomization.yaml
│       └── overlays/
│           ├── dev/
│           │   ├── kustomization.yaml
│           │   └── patches/
│           ├── staging/
│           │   ├── kustomization.yaml
│           │   └── patches/
│           └── production/
│               ├── kustomization.yaml
│               └── patches/
└── argocd/
    ├── projects/
    └── applications/
```

**base/kustomization.yaml:**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml
- configmap.yaml

commonLabels:
  app.kubernetes.io/name: myapp
  app.kubernetes.io/managed-by: argocd
```

**overlays/production/kustomization.yaml:**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: myapp-production

resources:
- ../../base

patchesStrategicMerge:
- patches/deployment-replicas.yaml
- patches/resource-limits.yaml

images:
- name: myapp
  newTag: v2.0.0

configMapGenerator:
- name: myapp-config
  behavior: merge
  literals:
  - LOG_LEVEL=info
  - ENVIRONMENT=production
```

**patches/deployment-replicas.yaml:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 5
```

**Applications por entorno:**

```yaml
# argocd/applications/myapp-dev.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-dev
  namespace: openshift-gitops
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/gitops-repo.git
    targetRevision: main
    path: apps/myapp/overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
---
# argocd/applications/myapp-prod.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-prod
  namespace: openshift-gitops
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/gitops-repo.git
    targetRevision: main
    path: apps/myapp/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp-production
  syncPolicy:
    automated:
      prune: false  # Manual en producción
      selfHeal: false
    syncOptions:
    - CreateNamespace=true
```

---

### Estrategias de promoción entre entornos

**Promoción con GitOps:**

```yaml
# .github/workflows/promote.yaml
name: Promote to Production

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to promote'
        required: true

jobs:
  promote:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
      with:
        repository: myorg/gitops-repo
        token: ${{ secrets.GITOPS_TOKEN }}
    
    - name: Update production image tag
      run: |
        cd apps/myapp/overlays/production
        kustomize edit set image myapp=myregistry.com/myapp:${{ github.event.inputs.version }}
    
    - name: Commit and push
      run: |
        git config user.name "CI Bot"
        git config user.email "ci@example.com"
        git add .
        git commit -m "Promote myapp to ${{ github.event.inputs.version }}"
        git push
```

**Promoción con ArgoCD Image Updater:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-staging
  annotations:
    argocd-image-updater.argoproj.io/image-list: myapp=myregistry.com/myapp
    argocd-image-updater.argoproj.io/myapp.update-strategy: semver
    argocd-image-updater.argoproj.io/myapp.allow-tags: regexp:^v1\..*
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/git-branch: main
spec:
  # ... resto de spec
```

---

## 4.5 Observabilidad

### Monitoring con Prometheus

OpenShift incluye Prometheus, Alertmanager y Grafana preinstalados.

**Acceder a Prometheus:**

```bash
# Ver pods de monitoring
oc get pods -n openshift-monitoring

# Rutas de monitoring
oc get routes -n openshift-monitoring

# Acceder a Prometheus
oc get route prometheus-k8s -n openshift-monitoring -o jsonpath='{.spec.host}'
# https://prometheus-k8s-openshift-monitoring.apps.cluster.example.com
```

**Custom ServiceMonitor:**

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp-monitor
  labels:
    release: prometheus  # Label para que Prometheus lo detecte
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
  - port: metrics
    path: /metrics
    interval: 30s
    scrapeTimeout: 10s
```

**Custom PrometheusRule (alertas):**

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: myapp-alerts
  labels:
    release: prometheus
spec:
  groups:
  - name: myapp.rules
    rules:
    - alert: MyAppHighErrorRate
      expr: |
        sum(rate(http_requests_total{app="myapp",status=~"5.."}[5m])) 
        / 
        sum(rate(http_requests_total{app="myapp"}[5m])) > 0.1
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: High error rate on myapp
        description: "Error rate is {{ $value | humanizePercentage }}"
    
    - alert: MyAppPodCrashLooping
      expr: |
        rate(kube_pod_container_status_restarts_total{container="myapp"}[15m]) > 0.1
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: Pod myapp is crash looping
```

**Dashboard personalizado con Grafana:**

```yaml
apiVersion: integreatly.org/v1alpha1
kind: GrafanaDashboard
metadata:
  name: myapp-dashboard
  labels:
    app: grafana
spec:
  json: |
    {
      "dashboard": {
        "title": "MyApp Dashboard",
        "panels": [
          {
            "title": "Request Rate",
            "type": "graph",
            "targets": [
              {
                "expr": "sum(rate(http_requests_total{app=\"myapp\"}[5m]))",
                "legendFormat": "Requests/s"
              }
            ]
          },
          {
            "title": "Error Rate",
            "type": "graph",
            "targets": [
              {
                "expr": "sum(rate(http_requests_total{app=\"myapp\",status=~\"5..\"}[5m])) / sum(rate(http_requests_total{app=\"myapp\"}[5m]))",
                "legendFormat": "Error %"
              }
            ]
          }
        ]
      }
    }
```

---

### Logging con Elasticsearch/Fluentd/Kibana

Ya cubierto arriba en la sección de Operators.

**Consultar logs desde CLI:**

```bash
# Logs de un pod específico
oc logs deployment/myapp

# Logs de múltiples pods
oc logs -l app=myapp

# Logs anteriores (si el pod se reinició)
oc logs deployment/myapp --previous

# Logs con timestamp
oc logs deployment/myapp --timestamps

# Logs en tiempo real
oc logs -f deployment/myapp

# Logs desde Elasticsearch
oc exec -n openshift-logging elasticsearch-cdm-xxx-1 -- \
  curl -s -k -u "internal:$(oc get secret elasticsearch -n openshift-logging -o jsonpath='{.data.elasticsearch\.key}' | base64 -d)" \
  "https://localhost:9200/_search?q=app:myapp&size=10" | jq
```

---

### Distributed Tracing con Jaeger

```yaml
# Deploy Jaeger
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: myapp-jaeger
  namespace: observability
spec:
  strategy: production
  storage:
    type: elasticsearch
    options:
      es:
        server-urls: http://elasticsearch:9200
  ingress:
    enabled: true
    hosts:
    - jaeger.apps.cluster.example.com
```

**Instrumentar aplicación para tracing:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    metadata:
      annotations:
        sidecar.jaegertracing.io/inject: "true"  # Auto-inyectar sidecar Jaeger
    spec:
      containers:
      - name: myapp
        image: myapp:latest
        env:
        - name: JAEGER_AGENT_HOST
          valueFrom:
            fieldRef:
              fieldPath: status.hostIP
        - name: JAEGER_AGENT_PORT
          value: "6831"
        - name: JAEGER_SAMPLER_TYPE
          value: const
        - name: JAEGER_SAMPLER_PARAM
          value: "1"
```

---

## Ejercicio práctico integrador - Día 4

Despliega una aplicación con Helm, configura CI/CD y observabilidad:

```bash
# 1. Crear chart Helm
mkdir -p myapp-chart/templates
cd myapp-chart

# Chart.yaml
cat > Chart.yaml <<EOF
apiVersion: v2
name: myapp
version: 1.0.0
appVersion: "1.0.0"
EOF

# values.yaml
cat > values.yaml <<EOF
replicaCount: 2
image:
  repository: nginx
  tag: alpine
service:
  type: ClusterIP
  port: 80
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi
EOF

# templates/deployment.yaml
cat > templates/deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
  labels:
    app: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
      - name: app
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: 80
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
EOF

# templates/service.yaml
cat > templates/service.yaml <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}
spec:
  selector:
    app: {{ .Release.Name }}
  ports:
  - port: {{ .Values.service.port }}
    targetPort: 80
EOF

# 2. Validar chart
helm lint .

# 3. Instalar
oc new-project ejercicio-dia4
helm install myapp . -n ejercicio-dia4

# 4. Verificar
helm list
oc get all

# 5. Actualizar
helm upgrade myapp . -n ejercicio-dia4 --set replicaCount=3

# 6. Ver historial
helm history myapp

# 7. Crear ServiceMonitor
cat <<EOF | oc apply -f -
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp-monitor
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
  - port: 80
    path: /metrics
    interval: 30s
EOF

# 8. Crear alerta
cat <<EOF | oc apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: myapp-alerts
  labels:
    release: prometheus
spec:
  groups:
  - name: myapp.rules
    rules:
    - alert: MyAppDown
      expr: kube_pod_status_ready{pod=~"myapp-.*"} == 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: MyApp pod is not ready
EOF

# 9. Exponer con Route
oc expose service myapp

# 10. Limpiar
helm uninstall myapp -n ejercicio-dia4
oc delete project ejercicio-dia4
```

---

## Resumen del Día 4

**Conceptos clave:**
- Node Selectors = filtro simple de nodos por labels
- Taints/Tolerations = repeler/atraer Pods a nodos
- Pod Affinity/Anti-Affinity = co-locación y dispersión de Pods
- Priority Classes = prioridad de Pods para scheduling y preemption
- Helm = gestor de paquetes para Kubernetes (Charts, Releases, Repos)
- Operators = controladores que gestionan aplicaciones complejas
- OLM = gestiona ciclo de vida de Operators
- CVO = gestiona actualizaciones del clúster OpenShift
- MCO = gestiona configuración de nodos
- Tekton = pipelines CI/CD nativos de Kubernetes
- ArgoCD = GitOps para sincronización declarativa
- Kustomize = customización de manifiestos por entorno
- Prometheus = monitoring y alerting
- Grafana = dashboards y visualización
- Jaeger = distributed tracing

**Comandos esenciales del día:**
```bash
# Scheduling
oc label node
oc adm taint nodes
oc get priorityclass

# Helm
helm install/upgrade/rollback/uninstall
helm list/history
helm repo add/update
helm lint/package

# Operators
oc get packagemanifests
oc get clusterserviceversion
oc get subscription

# GitOps
oc get applications -n openshift-gitops
argocd app sync/list/get

# Monitoring
oc get routes -n openshift-monitoring
oc get servicemonitor
oc get prometheusrule
```

---

# Resumen General del Curso

**Día 1:** Fundamentos de contenedores, arquitectura de OpenShift, recursos básicos (Pods, Deployments, Services, ConfigMaps, Secrets, PVCs)

**Día 2:** Orquestación avanzada (Networking, Routes, NetworkPolicies, StatefulSets, DaemonSets, Jobs, HPA, RBAC, SCC, gestión de recursos)

**Día 3:** Plataforma OpenShift (Instalación, Console, CLI oc, Proyectos, ImageStreams, S2I, BuildConfig, Routes avanzadas, OAuth, certificados)

**Día 4:** Plataforma avanzada (Scheduling, Helm, Operators, OLM, CVO, MCO, CI/CD, GitOps, Observabilidad)

¡El curso está completo! 🚀
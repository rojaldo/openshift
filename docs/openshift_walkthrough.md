# Walkthrough OpenShift - Guía de Demostración

## Objetivo
Mostrar las capacidades diferenciadoras de OpenShift respecto a Kubernetes vanilla mediante una serie de comandos prácticos con el cliente `oc`.

---

## Prerrequisitos

- OpenShift CLI (`oc`) instalado
- Acceso a un clúster OpenShift
- Usuario con permisos de administrador o desarrollador

### Verificar conexión

```bash
# Verificar versión del cliente
oc version

# Verificar conexión al clúster
oc status

# Verificar usuario actual
oc whoami
```

---

## 1. Proyectos (Namespaces con seguridad mejorada)

OpenShift usa "Projects" en lugar de namespaces simples. Los proyectos incluyen:
- Seguridad por defecto (SCC)
- Quotas y límites preconfigurados
- Aislamiento de red automático

```bash
# Listar proyectos
oc get projects

# Crear un proyecto (crea namespace + seguridad)
oc new-project demo-walkthrough \
  --display-name="Demo OpenShift" \
  --description="Walkthrough de capacidades OpenShift"

# Ver detalles del proyecto
oc describe project demo-walkthrough

# Ver quotas del proyecto
oc get quota -n demo-walkthrough

# Ver límites de recursos
oc get limits -n demo-walkthrough
```

**Diferencia con Kubernetes**: Los proyectos automáticamente configuran:
- Network policies
- Resource quotas por defecto
- Security context constraints
- Límites de recursos predefinidos

---

## 2. Security Context Constraints (SCC)

Una de las mayores diferencias: OpenShift no permite pods con root por defecto.

```bash
# Listar SCCs disponibles
oc get scc

# Ver SCC por defecto (restricted)
oc describe scc restricted

# Ver SCC para pods privilegiados
oc describe scc privileged

# Ver SCC para AnyUID (permite cualquier UID)
oc describe scc anyuid

# Añadir SCC a un service account
oc adm policy add-scc-to-user anyuid -z default -n demo-walkthrough

# Ver service accounts
oc get sa
```

**Diferencia con Kubernetes**:
- Kubernetes: Pods corren como root por defecto
- OpenShift: Pods corren con UID aleatorio, sin privilegios
- SCCs restringen capabilities, SELinux context, volumes permitidos

### Demo: Ver cómo OpenShift modifica los pods

```bash
# Crear un deployment simple
oc create deployment nginx --image=nginx

# Ver el pod creado
oc get pods -l app=nginx

# Inspeccionar el pod (nota el UID aleatorio)
oc get pod -l app=nginx -o yaml | grep -A5 "securityContext"

# Ver el UID asignado
oc get pod -l app=nginx -o jsonpath='{.items[0].spec.securityContext.runAsUser}'
```

---

## 3. Source-to-Image (S2I)

OpenShift puede construir imágenes desde código fuente directamente.

```bash
# Importar una imagen base
oc import-image openshift/python:3.9 --from=centos/python-39-centos7 --confirm

# Crear una aplicación desde código fuente
oc new-app python:3.9~https://github.com/sclorg/django-ex.git \
  --name=django-app

# Ver el build en progreso
oc logs -f bc/django-app

# Ver builds completados
oc get builds

# Ver la imagen construida
oc get is

# Ver el deployment resultante
oc get dc
```

**Diferencia con Kubernetes**:
- Kubernetes: Necesitas un pipeline CI/CD externo
- OpenShift: Construye imágenes desde código automáticamente con S2I

### Ver detalles del BuildConfig

```bash
# Ver el BuildConfig generado
oc get bc django-app -o yaml

# Ver ImageStreams
oc get is

# Ver DeploymentConfig
oc get dc django-app -o yaml
```

---

## 4. DeploymentConfigs vs Deployments

OpenShift tiene DeploymentConfigs con capacidades adicionales.

```bash
# Ver DeploymentConfigs
oc get dc

# Ver historial de despliegues
oc rollout history dc/django-app

# Ver revisiones
oc rollout history dc/django-app --revision=1

# Deshacer un despliegue
oc rollout undo dc/django-app

# Configurar triggers automáticos
oc set triggers dc/django-app --from-image=django-app:latest -c django-app

# Ver estrategias de deployment
oc get dc django-app -o jsonpath='{.spec.strategy.type}'

# Ver parámetros de rolling update
oc get dc django-app -o jsonpath='{.spec.strategy.rollingParams}'
```

**Diferencia con Kubernetes**:
- DeploymentConfigs tienen hooks (pre, post, mid)
- Soporte para triggers automáticos (image change, config change)
- Rollbacks con historial completo
- Estrategias: Rolling, Recreate, Custom

### Demo: Hooks en DeploymentConfig

```bash
# Añadir hook de pre-deployment
oc patch dc/django-app -p '{"spec":{"strategy":{"rollingParams":{"pre":{"execNewPod":{"command":["/bin/sh","-c","echo Pre-hook running"],"containerName":"django-app"},"failurePolicy":"Abort"}}}}}'

# Ver la configuración del hook
oc get dc django-app -o yaml | grep -A10 "pre:"
```

---

## 5. Routes vs Ingress

OpenShift usa Routes como equivalente a Ingress con capacidades adicionales.

```bash
# Crear una ruta simple
oc expose svc django-app --hostname=django.apps.example.com

# Ver rutas
oc get routes

# Ver detalles de la ruta
oc describe route django-app

# Crear ruta con TLS
oc create route edge django-app-tls \
  --service=django-app \
  --hostname=django-secure.apps.example.com

# Ver certificado generado
oc get route django-app-tls -o yaml | grep -A10 "tls:"

# Crear ruta con path-based routing
oc create route edge api-route \
  --service=api-service \
  --path=/api \
  --hostname=apps.example.com
```

**Diferencia con Kubernetes Ingress**:
- Routes generan certificados TLS automáticamente
- Soporte nativo para edge termination
- Integración con OpenShift Router (HAProxy)
- Path rewriting automático

### Ver Router configuration

```bash
# Ver el router (controller de OpenShift)
oc get pods -n openshift-ingress

# Ver configuración del router
oc get ingresscontroller default -n openshift-ingress-operator -o yaml

# Ver métricas del router
oc get route -n openshift-ingress
```

---

## 6. ImageStreams

OpenShift gestiona imágenes como recursos nativos.

```bash
# Ver ImageStreams
oc get is

# Ver tags disponibles
oc describe is nginx

# Importar imagen externa
oc import-image my-nginx:latest --from=nginx:latest --confirm

# Configurar importación automática
oc tag --source=docker nginx:latest my-nginx:latest

# Ver historial de tags
oc describe is my-nginx

# Crear ImageStreamTag manual
oc tag my-nginx:latest my-nginx:stable

# Ver referencias de imagen
oc get istag
```

**Diferencia con Kubernetes**:
- Kubernetes: Usa directamente el nombre de imagen del registry
- OpenShift: Abstrae las imágenes con ImageStreams
- Permite tags internos, triggers automáticos
- Historial de imágenes, rollback de imágenes

---

## 7. ConfigMaps y Secrets (con cifrado)

OpenShift cifra secrets por defecto.

```bash
# Crear un secret
oc create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=secret123

# Ver secrets
oc get secrets

# Montar secret en un deployment
oc set volume dc/django-app --add \
  --secret-name=db-secret \
  --mount-path=/etc/secrets

# Ver que está cifrado
oc get secret db-secret -o yaml | grep data:

# Crear secret desde archivo
oc create secret generic tls-secret \
  --from-file=tls.crt=/path/to/cert \
  --from-file=tls.key=/path/to/key

# Ver cifrado en etcd (necesita acceso admin)
oc get secret db-secret -o json | jq -r '.data.password' | base64 -d
```

**Diferencia con Kubernetes**:
- OpenShift cifra secrets en etcd por defecto
- Integración con Vault externo
- Secrets montados como archivos temporales

---

## 8. BuildConfigs y Pipeline Integrado

OpenShift tiene un sistema de builds nativo.

```bash
# Ver BuildConfigs
oc get bc

# Ver builds
oc get builds

# Iniciar un build manual
oc start-build django-app

# Ver logs del build
oc logs build/django-app-1

# Cancelar un build
oc cancel-build django-app-2

# Configurar webhook para builds automáticos
oc get bc django-app -o jsonpath='{.spec.triggers}'

# Crear BuildConfig desde Dockerfile
cat <<EOF | oc apply -f -
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: custom-build
spec:
  source:
    dockerfile: |
      FROM centos:7
      RUN yum install -y httpd
      CMD ["httpd", "-D", "FOREGROUND"]
  strategy:
    type: Docker
  output:
    to:
      kind: ImageStreamTag
      name: custom-image:latest
EOF
```

**Diferencia con Kubernetes**:
- Builds integrados en el clúster
- Múltiples estrategias: Docker, S2I, Pipeline, Custom
- Webhooks automáticos para CI/CD
- Historial de builds completo

---

## 9. Operadores y OperatorHub

OpenShift incluye un catálogo de operadores.

```bash
# Ver operadores disponibles
oc get packagemanifests -n openshift-marketplace | head -20

# Buscar un operador
oc search postgresql

# Ver detalles de un operador
oc describe packagemanifest postgresql -n openshift-marketplace

# Instalar un operador (ejemplo)
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: postgresql-operator
  namespace: openshift-operators
spec:
  channel: stable
  name: postgresql
  source: community-operators
  sourceNamespace: openshift-marketplace
EOF

# Ver operadores instalados
oc get subscriptions -n openshift-operators

# Ver clusterserviceversions
oc get csvs -n openshift-operators

# Ver CRDs disponibles
oc get crd | grep postgresql
```

**Diferencia con Kubernetes**:
- OperatorHub integrado
- Instalación con un clic desde la consola
- Gestión del ciclo de vida automática
- Actualizaciones automáticas de operadores

---

## 10. Monitoreo Integrado

OpenShift incluye Prometheus, AlertManager y Grafana.

```bash
# Ver el stack de monitoreo
oc get pods -n openshift-monitoring

# Ver Prometheus
oc get prometheus -n openshift-monitoring

# Ver AlertManager
oc get alertmanager -n openshift-monitoring

# Acceder a Prometheus (port-forward)
oc port-forward -n openshift-monitoring svc/prometheus-operated 9090:9090

# Ver alertas configuradas
oc get prometheusrules -n openshift-monitoring

# Ver métricas del clúster
oc adm top nodes

# Ver uso de recursos por pods
oc adm top pods -n demo-walkthrough

# Ver configuración de monitoreo
oc get configmap cluster-monitoring-config -n openshift-monitoring
```

**Diferencia con Kubernetes**:
- Stack completo instalado por defecto
- Métricas de OpenShift específicas
- Alertas preconfiguradas
- Grafana dashboards incluidos

---

## 11. Logging Agregado

OpenShift tiene EFK (Elasticsearch, Fluentd, Kibana) integrado.

```bash
# Ver pods de logging
oc get pods -n openshift-logging

# Ver Elasticsearch
oc get elasticsearch -n openshift-logging

# Ver Kibana
oc get kibana -n openshift-logging

# Ver Fluentd (collector)
oc get daemonset fluentd -n openshift-logging

# Ver logs de un pod específico
oc logs deployment/django-app

# Ver logs agregados (si está configurado)
oc logs deployment/django-app --all-namespaces

# Streaming de logs
oc logs -f deployment/django-app
```

---

## 12. Red y Seguridad de Red

OpenShift incluye NetworkPolicies y SDN por defecto.

```bash
# Ver configuración de red
oc get network.config.openshift.io cluster -o yaml

# Ver plugin de red
oc get network.config.openshift.io cluster -o jsonpath='{.spec.networkType}'

# Ver NetworkPolicies
oc get networkpolicy

# Crear NetworkPolicy
cat <<EOF | oc apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-openshift-ingress
spec:
  podSelector: {}
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          network.openshift.io/policy-group: ingress
EOF

# Ver proyectos aislados
oc get netnamespace

# Ver configuración de Egress
oc get egressnetworkpolicy
```

**Diferencia con Kubernetes**:
- SDN configurado por defecto
- Proyectos aislados automáticamente
- Egress network policies (solo OpenShift)
- Multitenant isolation

---

## 13. Templates y QuickStarts

OpenShift permite crear aplicaciones desde templates.

```bash
# Listar templates disponibles
oc get templates -n openshift

# Ver un template específico
oc describe template postgresql-ephemeral -n openshift

# Procesar un template (sin crear)
oc process openshift//postgresql-ephemeral \
  -p POSTGRESQL_USER=myuser \
  -p POSTGRESQL_PASSWORD=mypassword \
  -p POSTGRESQL_DATABASE=mydb \
  --dry-run -o yaml

# Crear desde template
oc new-app --template=postgresql-ephemeral \
  -p POSTGRESQL_USER=myuser \
  -p POSTGRESQL_PASSWORD=mypassword \
  -p POSTGRESQL_DATABASE=mydb

# Crear template personalizado
oc create -f - <<EOF
apiVersion: template.openshift.io/v1
kind: Template
metadata:
  name: my-template
objects:
- apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: \${NAME}
  spec:
    replicas: \${REPLICAS}
    template:
      metadata:
        labels:
          app: \${NAME}
      spec:
        containers:
        - name: \${NAME}
          image: \${IMAGE}
parameters:
- name: NAME
  value: myapp
- name: REPLICAS
  value: "1"
- name: IMAGE
  value: nginx:latest
EOF

# Usar el template
oc process my-template -p NAME=webapp -p REPLICAS=3 | oc apply -f -
```

---

## 14. Role-Based Access Control (RBAC)

OpenShift extiende RBAC con roles específicos.

```bash
# Ver roles del clúster
oc get clusterroles | head -20

# Ver roles del proyecto
oc get roles

# Ver role bindings
oc get rolebindings

# Añadir rol a usuario
oc adm policy add-role-to-user admin usuario1 -n demo-walkthrough

# Añadir rol a grupo
oc adm policy add-role-to-group view developers -n demo-walkthrough

# Ver quién puede hacer qué
oc auth can-i --list

# Ver permisos de un usuario
oc auth can-i get pods --as=usuario1

# Ver grupos
oc get groups

# Crear grupo
oc adm groups new developers usuario1 usuario2
```

**Diferencia con Kubernetes**:
- Roles específicos de OpenShift: admin, edit, view
- Integración con LDAP/OAuth
- Grupos de usuarios nativos
- Project access control

---

## 15. Consola Web

La consola de OpenShift ofrece más funcionalidades.

```bash
# Obtener URL de la consola
oc whoami --show-console

# Obtener URL de la API
oc whoami --show-server

# Obtener token de acceso
oc whoami -t

# Login con token
oc login --token=<token> --server=<server>

# Ver info del usuario actual
oc whoami
oc get user $(oc whoami) -o yaml
```

---

## Limpieza

```bash
# Eliminar el proyecto de demo
oc delete project demo-walkthrough

# Eliminar operadores instalados
oc delete subscription postgresql-operator -n openshift-operators

# Eliminar CRDs
oc delete crd postgresqls.postgresql.k8s.io
```

---

## Resumen de Diferencias

| Característica | Kubernetes | OpenShift |
|---------------|------------|-----------|
| **Seguridad por defecto** | Pods como root | Random UIDs, SCCs |
| **Builds** | Externo (CI/CD) | Integrado (S2I, BuildConfig) |
| **Deployments** | Deployment | Deployment + DeploymentConfig |
| **Ingress** | Ingress Controller | Routes (con TLS automático) |
| **Imágenes** | Directas del registry | ImageStreams abstractos |
| **Templates** | Helm Charts | Templates nativos + Helm |
| **Operadores** | Manual | OperatorHub integrado |
| **Monitoreo** | Instalar manual | Stack completo incluido |
| **Logging** | Instalar manual | EFK incluido |
| **Red** | Plugin CNI | SDN por defecto + Egress |
| **RBAC** | Básico | Roles extendidos + grupos |
| **Consola** | Dashboard básico | Consola completa |

---


## 16. Consola Web

```bash
# Obtener URL de la consola
oc whoami --show-console

# Abrir consola (si estás en navegador)
open $(oc whoami --show-console)

# Navegación básica:
# - Home > Overview: Vista general del proyecto
# - Workloads > Deployments: Ver deployments
# - Workloads > Pods: Ver pods y logs
# - Networking > Routes: Ver rutas
# - Builds > Build Configs: Ver builds
# - Operators > OperatorHub: Instalar operadores
# - Monitoring > Metrics: Ver métricas
# - Monitoring > Alerts: Ver alertas
```

## 17. Developer Perspective

```bash
# Cambiar a vista de desarrollador en la consola
# URL: https://console-openshift-console.apps.<cluster>/topology

# Opciones disponibles:
# - Topology View: Ver aplicaciones gráficamente
# - Add > From Git: Desplegar desde repositorio
# - Add > Container Image: Desplegar imagen existente
# - Add > From Catalog: Usar operadores
# - Add > YAML: Crear recursos manualmente

# Comandos equivalentes:
oc get all -n demo-walkthrough
oc get pods -o wide
oc get routes
oc get svc
```

## 18. Admin Perspective

```bash
# Cambiar a vista de administrador
# URL: https://console-openshift-console.apps.<cluster>/overview

# Navegación de administrador:
# - Home > Overview: Estado del clúster
# - Home > Alerts: Alertas activas
# - Operators > Installed Operators: Operadores instalados
# - Operators > OperatorHub: Catálogo de operadores
# - Workloads > Jobs: Ver jobs del sistema
# - Networking > NetworkPolicies: Políticas de red
# - Storage > PersistentVolumes: Volúmenes persistentes
# - Compute > Nodes: Nodos del clúster
# - Compute > MachineSets: Machine sets
# - Administration > Cluster Settings: Configuración
# - Administration > Resource Quotas: Cuotas
# - Administration > LimitRanges: Límites

# Comandos equivalentes:
oc get nodes
oc get co  # Cluster Operators
oc get csr  # Certificate Signing Requests
oc get pv
oc get namespaces
oc get clusteroperators
```

## 19. Monitoring y Dashboards

```bash
# Acceder a Grafana
oc get route grafana -n openshift-monitoring -o jsonpath='{.spec.host}'

# Abrir Grafana
open "https://$(oc get route grafana -n openshift-monitoring -o jsonpath='{.spec.host}')"

# Dashboards disponibles:
# - Kubernetes/Compute Resources/Cluster
# - Kubernetes/Compute Resources/Namespace(Pods)
# - OpenShift/Kubernetes/API Server
# - OpenShift/Kubernetes/Controller Manager
# - USE Method/Cluster
# - USE Method/Node

# Ver métricas desde CLI
oc adm top nodes
oc adm top pods -A

# Ver métricas de Prometheus
oc port-forward -n openshift-monitoring svc/prometheus-operated 9090:9090
# Abrir: http://localhost:9090

# Queries útiles:
# - up{job="prometheus"}
# - cluster:cpu_usage:ratio
# - cluster:memory_usage:ratio
# - namespace:container_memory_usage_bytes:sum
```

## 20. AlertManager y Alertas

```bash
# Acceder a AlertManager
oc get route alertmanager-main -n openshift-monitoring -o jsonpath='{.spec.host}'

# Abrir AlertManager
open "https://$(oc get route alertmanager-main -n openshift-monitoring -o jsonpath='{.spec.host}')"

# Ver alertas activas
oc get alertmanagers -n openshift-monitoring

# Ver reglas de alerta
oc get prometheusrules -n openshift-monitoring

# Crear alerta personalizada
oc apply -f - <<EOF
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: my-custom-alerts
  namespace: demo-walkthrough
spec:
  groups:
  - name: custom-alerts
    rules:
    - alert: HighMemoryUsage
      expr: container_memory_usage_bytes{namespace="demo-walkthrough"} > 1073741824
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High memory usage detected"
        description: "Container is using more than 1GB memory"
EOF

# Ver alertas configuradas
oc get prometheusrules -n demo-walkthrough -o yaml
```

## 21. Explorar OperatorHub

```bash
# Ver todos los operadores disponibles
oc get packagemanifests -n openshift-marketplace --sort-by='.metadata.creationTimestamp'

# Buscar operadores por categoría
oc get packagemanifests -n openshift-marketplace -l catalog=community-operators

# Ver detalles de un operador
oc describe packagemanifest postgresql-operator -n openshift-marketplace

# Ver canales disponibles
oc get packagemanifest postgresql-operator -n openshift-marketplace -o jsonpath='{.status.channels[*].name}'

# Ver versiones disponibles
oc get packagemanifest postgresql-operator -n openshift-marketplace -o yaml | grep -A5 "version:"
```

## 22. Service Mesh (OpenShift Service Mesh)

```bash
# Verificar si Service Mesh está instalado
oc get servicemeshcontrolplane -n istio-system

# Ver operadores de Service Mesh
oc get csvs -n openshift-operators | grep mesh

# Crear Service Mesh Control Plane
cat <<EOF | oc apply -f -
apiVersion: maistra.io/v1
kind: ServiceMeshControlPlane
metadata:
  name: basic
  namespace: istio-system
spec:
  version: v2.0
  tracing:
    type: Jaeger
    sampling: 10000
  policy:
    type: Istiod
  addons:
    grafana:
      enabled: true
    kiali:
      enabled: true
    prometheus:
      enabled: true
EOF

# Verificar instalación
oc get pods -n istio-system

# Acceder a Kiali dashboard
oc get route kiali -n istio-system -o jsonpath='{.spec.host}'
```

## 23. Serverless (OpenShift Serverless)

```bash
# Verificar si Serverless está instalado
oc get knativeserving -n knative-serving

# Crear función serverless
cat <<EOF | oc apply -f -
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: hello-serverless
  namespace: demo-walkthrough
spec:
  template:
    spec:
      containers:
      - image: gcr.io/knative-samples/helloworld-go
        env:
        - name: TARGET
          value: "OpenShift Serverless"
EOF

# Ver servicio serverless
oc get ksvc hello-serverless -n demo-walkthrough

# Ver revisiones
oc get revisions -n demo-walkthrough

# Escalar a cero (scale to zero)
oc patch ksvc hello-serverless -n demo-walkthrough -p '{"spec":{"template":{"spec":{"traffic":[{"percent":100,"revisionName":"hello-serverless-00001"}]}}}}'
```

## 24. Pipelines (OpenShift Pipelines / Tekton)

```bash
# Verificar si Pipelines está instalado
oc get tektonconfig config

# Ver pipelines disponibles
oc get pipelines -n demo-walkthrough

# Crear pipeline simple
cat <<EOF | oc apply -f -
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: build-and-deploy
  namespace: demo-walkthrough
spec:
  params:
  - name: git-url
    type: string
  - name: image-name
    type: string
  tasks:
  - name: fetch-repo
    taskRef:
      name: git-clone
    params:
    - name: url
      value: \$(params.git-url)
  - name: build-image
    taskRef:
      name: s2i
    params:
    - name: IMAGE
      value: \$(params.image-name)
EOF

# Ejecutar pipeline
oc create -f - <<EOF
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: build-and-deploy-run
  namespace: demo-walkthrough
spec:
  pipelineRef:
    name: build-and-deploy
  params:
  - name: git-url
    value: https://github.com/example/app.git
  - name: image-name
    value: image-registry.openshift-image-registry.svc:5000/demo-walkthrough/app:latest
EOF

# Ver pipeline runs
oc get pipelineruns -n demo-walkthrough
```

## 25. Helm Charts en OpenShift

```bash
# Ver repositorios Helm
oc get helmchartrepository -A

# Añadir repositorio Helm
cat <<EOF | oc apply -f -
apiVersion: helm.openshift.io/v1beta1
kind: HelmChartRepository
metadata:
  name: bitnami-charts
spec:
  name: bitnami
  connectionConfig:
    url: https://charts.bitnami.com/bitnami
EOF

# Buscar charts
oc get helmchartrepository bitnami-charts -o yaml

# Instalar chart desde consola:
# Developer > Add > Helm Chart > Seleccionar chart > Install
```

## 26. Persistent Storage

```bash
# Ver storage classes disponibles
oc get storageclasses

# Ver persistent volumes
oc get pv

# Ver persistent volume claims
oc get pvc -n demo-walkthrough

# Crear PVC dinámico
cat <<EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
  namespace: demo-walkthrough
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard
EOF

# Ver estado del PVC
oc describe pvc my-pvc -n demo-walkthrough

# Montar PVC en deployment
oc set volume dc/django-app --add --claim=my-pvc --mount-path=/data
```

## 27. Configuración de Cluster

```bash
# Ver configuración del clúster
oc get clusterversion

# Ver operadores del clúster
oc get clusteroperators

# Ver infraestructura
oc get infrastructures.config.openshift.io cluster -o yaml

# Ver red del clúster
oc get network.config.openshift.io cluster -o yaml

# Ver versiones de API disponibles
oc api-resources | head -30

# Ver recursos personalizados
oc get crds | wc -l

# Ver configuración de autenticación
oc get oauth cluster -o yaml

# Ver identity providers
oc get identityproviders
```

## 28. Backup y Restore

```bash
# Ver backups disponibles (si está instalado)
oc get backups -A

# Ver schedules de backup
oc get schedules -A

# Crear backup de recursos
oc get all,configmaps,secrets -n demo-walkthrough -o yaml > backup-demo.yaml

# Restaurar desde backup
oc apply -f backup-demo.yaml
```

## 29. Escalado Automático

```bash
# Ver Horizontal Pod Autoscalers
oc get hpa -n demo-walkthrough

# Crear HPA
oc autoscale dc/django-app --min=1 --max=10 --cpu-percent=80

# Ver HPA estado
oc get hpa django-app -n demo-walkthrough

# Ver métricas para HPA
oc describe hpa django-app -n demo-walkthrough

# Cluster Autoscaler (admin)
oc get clusterAutoscaler -A

# Machine Autoscaler (admin)
oc get machineAutoscaler -A
```

## 30. Debug y Troubleshooting

```bash
# Debug de nodos
oc debug node/<node-name>

# Debug de pods
oc debug pod/<pod-name>

# Ver eventos del namespace
oc get events -n demo-walkthrough --sort-by='.metadata.creationTimestamp'

# Ver eventos del clúster
oc get events -A --field-selector reason=Failed

# Ver logs de componentes del sistema
oc logs -n openshift-apiserver deployment/apiserver
oc logs -n openshift-controller-manager deployment/controller-manager

# Ver estado de componentes
oc get co

# Diagnosticar problemas de nodo
oc adm top nodes
oc describe node <node-name>

# Diagnosticar problemas de pod
oc describe pod <pod-name>
oc logs <pod-name> --previous  # Logs del container anterior
oc logs <pod-name> -c <container-name>  # Logs de container específico
```

## 31. Resource Quotas y Limits

```bash
# Ver quotas del clúster
oc get clusterresourcequota

# Crear quota para proyecto
oc create quota project-quota --hard=cpu=4,memory=8Gi,pods=10 -n demo-walkthrough

# Ver quota
oc describe quota project-quota -n demo-walkthrough

# Ver limit ranges
oc get limits -n demo-walkthrough

# Crear limit range
cat <<EOF | oc apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: demo-walkthrough
spec:
  limits:
  - type: Container
    max:
      cpu: "2"
      memory: 4Gi
    min:
      cpu: 100m
      memory: 128Mi
    default:
      cpu: "1"
      memory: 1Gi
    defaultRequest:
      cpu: 200m
      memory: 256Mi
EOF
```

## 32. Secrets y ConfigMaps

```bash
# Ver secrets
oc get secrets -n demo-walkthrough

# Crear secret desde literal
oc create secret generic app-secret --from-literal=db-password=secret123 -n demo-walkthrough

# Crear secret desde archivo
oc create secret generic tls-secret --from-file=tls.crt=/path/to/cert --from-file=tls.key=/path/to/key -n demo-walkthrough

# Ver configmaps
oc get configmaps -n demo-walkthrough

# Crear configmap
oc create configmap app-config --from-literal=APP_ENV=production --from-literal=LOG_LEVEL=info -n demo-walkthrough

# Montar configmap como volumen
oc set volume dc/django-app --add --configmap-name=app-config --mount-path=/etc/config
```

## 33. Network Policies

```bash
# Ver network policies
oc get networkpolicy -n demo-walkthrough

# Crear network policy para denegar todo ingress
cat <<EOF | oc apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: demo-walkthrough
spec:
  podSelector: {}
  policyTypes:
  - Ingress
EOF

# Crear network policy para permitir tráfico desde routes
cat <<EOF | oc apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-openshift-ingress
  namespace: demo-walkthrough
spec:
  podSelector: {}
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          network.openshift.io/policy-group: ingress
EOF

# Ver netnamespace (OpenShift específico)
oc get netnamespace demo-walkthrough

# Ver hostsubnet (admin)
oc get hostsubnets
```

## 34. CronJobs y Jobs

```bash
# Ver cronjobs
oc get cronjobs -n demo-walkthrough

# Crear cronjob
cat <<EOF | oc apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-job
  namespace: demo-walkthrough
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: busybox
            command:
            - /bin/sh
            - -c
            - echo "Running backup at $(date)"
          restartPolicy: OnFailure
EOF

# Ver jobs
oc get jobs -n demo-walkthrough

# Crear job manual
oc create job manual-backup --from=cronjob/backup-job -n demo-walkthrough
```

## 35. Limpieza Final

```bash
# Eliminar proyecto completo
oc delete project demo-walkthrough

# Eliminar todos los recursos de un tipo
oc delete all -l app=django-app -n demo-walkthrough

# Eliminar recursos huérfanos
oc get pvc -n demo-walkthrough --no-headers | awk '{print $1}' | xargs oc delete pvc -n demo-walkthrough

# Limpiar builds antiguos
oc adm prune builds --keep-complete=5 --keep-failed=3

# Limpiar imágenes antiguas
oc adm prune images --keep-tag-revisions=3

# Limpiar deployments antiguos
oc adm prune deployments --keep-complete=5 --keep-failed=3
```

---

## Referencias

- [Documentación OpenShift](https://docs.openshift.com)
- [CLI Reference](https://docs.openshift.com/container-platform/4.14/cli_reference/oc-cli-developer-commands.html)
- [Security Context Constraints](https://docs.openshift.com/container-platform/4.14/authentication/managing-security-context-constraints.html)
- [Source-to-Image](https://docs.openshift.com/container-platform/4.14/builds/build-strategies.html#build-strategy-s2i_build-strategies)
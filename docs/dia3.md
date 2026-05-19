# OpenShift Container Platform - Día 3
## OpenShift Container Platform

---

## 3.1 Introducción a OpenShift Container Platform

### OpenShift como distribución empresarial: OKD vs OCP

**OCP (OpenShift Container Platform):**
- Versión comercial de Red Hat
- Soporte empresarial
- Certificaciones y compliance
- Acceso a Red Hat Registry
- Pago por suscripción

**OKD (Origin Community Distribution):**
- Versión upstream/community
- Gratis y open source
- Sin soporte comercial
- Base de Fedora CoreOS
- Idéntico funcionalmente a OCP

**Comparación:**

| Aspecto | OKD | OCP |
|---------|-----|-----|
| Licencia | Apache 2.0 | Comercial |
| Soporte | Comunidad | Red Hat |
| Registry | Quay público | registry.redhat.io |
| Actualizaciones | Manual | Automáticas con CVO |
| Enterprise ready | Limitado | Full compliance |

**¿Cuándo usar cada uno?**
- **OCP:** Producción empresarial, compliance, soporte 24/7
- **OKD:** Desarrollo, testing, learning, proyectos sin budget

---

### Arquitectura de OpenShift Container Platform

OpenShift añade capas sobre Kubernetes:

```
┌─────────────────────────────────────────────────────┐
│              OpenShift Platform Services            │
│  ┌────────────┐ ┌────────────┐ ┌────────────────┐ │
│  │   Router   │ │   OAuth    │ │ Image Registry │ │
│  │  (HAProxy) │ │   Server   │ │   (Integrated) │ │
│  └────────────┘ └────────────┘ └────────────────┘ │
│  ┌────────────┐ ┌────────────┐ ┌────────────────┐ │
│  │   Web      │ │  Operator  │ │   Monitoring   │ │
│  │   Console  │ │   Hub      │ │   (Prometheus) │ │
│  └────────────┘ └────────────┘ └────────────────┘ │
├─────────────────────────────────────────────────────┤
│                  Kubernetes Core                    │
│  ┌──────────┐ ┌──────────┐ ┌───────────────────┐  │
│  │   API    │ │  etcd    │ │    Scheduler      │  │
│  │  Server  │ │          │ │                   │  │
│  └──────────┘ └──────────┘ └───────────────────┘  │
│  ┌──────────────────────────────────────────────┐ │
│  │              Controller Manager               │ │
│  └──────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────┤
│                 Node Components                     │
│  ┌──────────┐ ┌──────────┐ ┌───────────────────┐  │
│  │ kubelet  │ │kube-proxy│ │     CRI-O         │  │
│  └──────────┘ └──────────┘ └───────────────────┘  │
│  ┌──────────┐ ┌──────────┐ ┌───────────────────┐  │
│  │   OVN    │ │  MCO     │ │   Node Exporter   │  │
│  │ Kubernetes│ │ Daemon   │ │                   │  │
│  └──────────┘ └──────────┘ └───────────────────┘  │
├─────────────────────────────────────────────────────┤
│              Red Hat Enterprise Linux               │
│              CoreOS (RHCOS) / Fedora CoreOS         │
└─────────────────────────────────────────────────────┘
```

**Componentes únicos de OpenShift:**

**Router (Ingress Controller):**
- Basado en HAProxy
- Gestiona Routes (equivalente a Ingress)
- Balanceo de carga L7
- TLS termination

**Integrated Registry:**
- Registry de imágenes interno
- Almacenamiento persistente
- Image streams automáticos

**OAuth Server:**
- Autenticación integrada
- Proveedores: HTPasswd, LDAP, GitHub, Google, OIDC

**Web Console:**
- UI para administradores y desarrolladores
- Topología visual de aplicaciones
- Terminal web embebida

**Operators:**
- Lifecycle management automatizado
- OperatorHub con catálogo
- OLM (Operator Lifecycle Manager)

---

### Opciones de instalación: IPI, UPI, SNO, OpenShift Local

**IPI (Installer-Provided Infrastructure):**
- Instalación automatizada
- Provisioning de infraestructura (VMs, redes, storage)
- Soportado: AWS, Azure, GCP, vSphere, Bare Metal, Nutanix, IBM Cloud

```bash
# Instalación IPI en AWS
openshift-install create install-config --dir=my-cluster
# Editar install-config.yaml
openshift-install create cluster --dir=my-cluster
```

**UPI (User-Provided Infrastructure):**
- Usuario prepara infraestructura
- Más control
- Más complejo
- Para entornos específicos

```bash
# UPI requiere:
# - VMs provisionadas manualmente
# - Load balancers configurados
# - DNS configurado
# - Ignition configs generadas
openshift-install create ignition-configs --dir=my-cluster
```

**SNO (Single Node OpenShift):**
- Un solo nodo (control plane + workers)
- Para edge, development, labs
- Recursos mínimos: 8 vCPU, 16GB RAM, 100GB storage

```bash
# Instalación SNO
openshift-install create single-node-ignition-config --dir=sno-cluster
```

**OpenShift Local (antiguo CRC - CodeReady Containers):**
- Clúster local para desarrollo
- Corre en una VM en tu laptop
- Recursos: 4 vCPU, 9GB RAM mínimo

```bash
# Instalar OpenShift Local
# Descargar desde: https://console.redhat.com/openshift/create/local

# Configurar
crc setup

# Iniciar clúster
crc start
# Output: API URL, Web Console URL, kubeadmin password

# Configurar oc
eval $(crc oc-env)
oc login -u kubeadmin -p <password> https://api.crc.testing:6443

# Acceder a la consola
crc console

# Detener
crc stop

# Eliminar
crc delete
```

**Comparación de opciones:**

| Opción | Nodos | Uso | Complejidad |
|--------|-------|-----|-------------|
| IPI | 3+ | Producción | Media |
| UPI | 3+ | Producción personalizada | Alta |
| SNO | 1 | Edge, Labs | Baja |
| OpenShift Local | 1 | Development | Muy baja |

---

### Consola web: perspectiva de Administrador y de Desarrollador

**Perspectiva de Administrador:**
- Gestión de clúster completo
- Nodes, Projects, Workloads
- Networking, Storage, Security
- Operators installation
- Monitoring and alerts

**Perspectiva de Desarrollador:**
- Topología visual de aplicaciones
- Deployments, Builds, Pipelines
- Project-scoped view
- Code import from Git
- Quick create from catalog

**Acceso:**

```bash
# Obtener URL de consola
oc whoami --show-console
# https://console-openshift-console.apps.cluster.example.com

# Obtener credenciales
oc get secret kubeadmin-password -n openshift-config \
  -o jsonpath='{.data.password}' | base64 -d
```

**Features clave:**

**Topología visual:**
- Muestra deployments, services, routes
- Conexiones entre componentes
- Click para detalles
- Escalar/editar desde UI

**Terminal web:**
```bash
# Disponible desde la consola
# Botón ">_" en la esquina superior derecha
# Terminal con oc ya configurado
```

**Developer Catalog:**
- Templates de aplicaciones
- Databases, middleware
- Operators disponibles

---

### CLI oc: comandos esenciales

`oc` es el CLI de OpenShift, superconjunto de `kubectl`.

**Comandos básicos:**

```bash
# Login
oc login -u kubeadmin -p <password> https://api.cluster.example.com:6443
oc login -u developer -p developer https://api.crc.testing:6443

# Logout
oc logout

# Ver usuario actual
oc whoami

# Ver contexto
oc config current-context

# Cambiar de proyecto
oc project <nombre>
oc project  # ver proyecto actual

# Listar proyectos
oc projects

# Crear proyecto
oc new-project mi-app --description="Mi aplicación"
```

**Trabajo con recursos:**

```bash
# Ver recursos (similar a kubectl)
oc get pods
oc get deployments
oc get services
oc get routes
oc get all

# Ver todos los recursos de un tipo
oc get pods -A  # todos los namespaces
oc get pods -n openshift-apiserver

# Detalles
oc describe pod <nombre>
oc describe deployment <nombre>

# Crear desde YAML
oc apply -f deployment.yaml

# Editar recurso
oc edit deployment <nombre>

# Eliminar
oc delete pod <nombre>
oc delete deployment <nombre>
oc delete project <nombre>
```

**Comandos específicos de OpenShift:**

```bash
# new-app: crear aplicación completa
oc new-app nginx~https://github.com/user/repo.git
oc new-app --name=webapp nginx:alpine
oc new-app --docker-image=quay.io/myapp:latest

# Exponer servicio
oc expose service webapp
oc expose service webapp --hostname=webapp.apps.mycluster.com

# Ver logs
oc logs deployment/webapp
oc logs deployment/webapp -c container-name
oc logs -f deployment/webapp  # follow

# Ejecutar comando en Pod
oc exec deployment/webapp -- ls /app
oc exec -it deployment/webapp -- sh

# Port-forward
oc port-forward pod/webapp-123 8080:80
oc port-forward service/webapp 8080:80

# Copiar archivos
oc cp local-file.txt pod/webapp-123:/tmp/
oc cp pod/webapp-123:/app/logs ./logs
```

**Gestión de imágenes y builds:**

```bash
# ImageStreams
oc get imagestreams
oc import-image myapp:latest --from=quay.io/myapp:latest
oc tag myapp:v1 myapp:latest

# BuildConfigs
oc get buildconfigs
oc start-build bc/myapp
oc logs build/myapp-1

# Build logs en tiempo real
oc start-build bc/myapp --follow
```

**Debug y troubleshooting:**

```bash
# Debug de Pod
oc debug pod/webapp-123
oc debug deployment/webapp

# Debug de nodo
oc debug node/worker-0

# Ver eventos
oc get events
oc get events --sort-by='.lastTimestamp'

# Ver recursos usados
oc adm top pods
oc adm top nodes

# Analizar problema
oc logs deployment/webapp --previous  # logs del container anterior
oc get pod webapp-123 -o yaml  # ver spec completo
```

**Comandos avanzados:**

```bash
# Rollout
oc rollout status deployment/webapp
oc rollout history deployment/webapp
oc rollout undo deployment/webapp
oc rollout restart deployment/webapp

# Escalar
oc scale deployment/webapp --replicas=5
oc autoscale deployment/webapp --min=2 --max=10 --cpu-percent=70

# Labels y annotations
oc label pod webapp-123 env=production
oc label pod webapp-123 env-  # eliminar label
oc annotate deployment webapp description="Mi app"

# Selectores
oc get pods -l app=webapp
oc get pods -l 'app in (webapp,backend)'
oc get pods -l app=webapp,env=prod
```

---

## 3.2 Proyectos y construcción de imágenes

### Proyectos en OpenShift: modelo multi-tenant sobre namespaces

Un **Proyecto** es un Namespace de Kubernetes con funcionalidades añadidas:

**Diferencias Project vs Namespace:**
- Aislamiento de red por defecto
- Quota pre-configurada
- Roles específicos (admin, edit, view)
- Self-service para desarrolladores

```bash
# Crear proyecto
oc new-project demo \
  --description="Proyecto de demostración" \
  --display-name="Demo Project"

# Ver proyectos
oc get projects

# Ver proyecto actual
oc project

# Cambiar de proyecto
oc project demo

# Ver detalles
oc describe project demo

# Editar proyecto
oc edit project demo

# Eliminar proyecto
oc delete project demo
```

**Permisos predefinidos:**

```bash
# Ver roles disponibles
oc describe clusterrole admin
oc describe clusterrole edit
oc describe clusterrole view

# Asignar roles
oc adm policy add-role-to-user admin usuario1
oc adm policy add-role-to-user edit usuario2
oc adm policy add-role-to-user view usuario3

# Ver bindings
oc get rolebindings
```

---

### ImageStreams y etiquetado de imágenes

Un **ImageStream** es una abstracción sobre imágenes de contenedor que permite:
- Seguimiento de tags
- Referencias a registros externos
- Triggers automáticos de deployment

```yaml
# imagestream.yaml
apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  name: myapp
spec:
  lookupPolicy:
    local: false  # true para buscar solo en registry local
  tags:
  - name: latest
    from:
      kind: DockerImage
      name: quay.io/myuser/myapp:latest
    importPolicy:
      scheduled: true  # Importar periódicamente
```

```bash
# Crear ImageStream desde imagen existente
oc import-image myapp:latest --from=quay.io/myuser/myapp:latest

# Crear ImageStream vacío
oc create imagestream myapp

# Ver ImageStreams
oc get imagestreams
oc describe is myapp

# Ver tags en un ImageStream
oc get imagestream myapp -o jsonpath='{.status.tags[*].tag}'

# Etiquetar imagen
oc tag quay.io/myuser/myapp:v1.0 myapp:v1.0
oc tag myapp:v1.0 myapp:production

# Eliminar tag
oc tag -d myapp:v1.0

# Importar periódicamente
oc import-image myapp:latest --scheduled
```

**Triggers de ImageStream:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  triggers:
  - type: ImageChange
    imageChangeParams:
      automatic: true
      containerNames:
      - web
      from:
        kind: ImageStreamTag
        name: myapp:latest
```

---

### Source-to-Image (S2I)

S2I es una herramienta que genera imágenes de contenedor desde código fuente sin Dockerfile.

**Cómo funciona:**
1. Descarga imagen base (builder image)
2. Inyecta código fuente
3. Ejecuta scripts de build
4. Genera imagen final

**Builder images disponibles:**
- `ruby` - Rails, Sinatra
- `python` - Django, Flask
- `nodejs` - Express, React
- `java` - Spring Boot, WildFly
- `dotnet` - .NET Core
- `php` - Laravel, Symfony
- `golang` - Go apps

**Ejemplo - Aplicación Node.js:**

```bash
# Crear app desde Git con S2I
oc new-app nodejs~https://github.com/sclorg/nodejs-ex.git \
  --name=nodejs-app

# Ver progreso del build
oc logs -f bc/nodejs-app

# Ver ImageStream creado
oc get is

# Exponer
oc expose svc/nodejs-app

# Ver URL
oc get route
```

**S2I con configuración:**

```yaml
# buildconfig-s2i.yaml
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: nodejs-app
spec:
  source:
    type: Git
    git:
      uri: https://github.com/sclorg/nodejs-ex.git
    contextDir: /
  strategy:
    type: Source
    sourceStrategy:
      from:
        kind: ImageStreamTag
        name: nodejs:latest
        namespace: openshift
      env:
      - name: NODE_ENV
        value: production
  output:
    to:
      kind: ImageStreamTag
      name: nodejs-app:latest
  triggers:
  - type: ConfigChange
  - type: ImageChange
  - type: GitHub
    github:
      secret: mi-webhook-secret
```

**Custom S2I builder:**

```dockerfile
# Dockerfile para builder image personalizado
FROM centos:7

# Scripts de S2I
COPY ./s2i/bin/ /usr/libexec/s2i/

# Runtime
RUN yum install -y python3 && yum clean all

# Label obligatorio
LABEL io.openshift.s2i.scripts-url="image:///usr/libexec/s2i"

# Script assemble (se ejecuta en build)
# /usr/libexec/s2i/assemble
# Script run (ejecuta la app)
# /usr/libexec/s2i/run
```

---

### BuildConfig: tipos de build (Source, Dockerfile, Binary)

**BuildConfig** define cómo construir imágenes.

**Tipos de build:**

**1. Source (S2I):** Ya visto arriba

**2. Dockerfile:**

```yaml
# buildconfig-docker.yaml
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: docker-build
spec:
  source:
    type: Git
    git:
      uri: https://github.com/user/myapp.git
    contextDir: /
  strategy:
    type: Docker
    dockerStrategy:
      dockerfilePath: Dockerfile
      buildArgs:
      - name: NODE_ENV
        value: production
  output:
    to:
      kind: ImageStreamTag
      name: myapp:latest
  triggers:
  - type: ConfigChange
  - type: GitHub
    github:
      secret: webhook-secret
```

**3. Binary:**

```bash
# Build desde directorio local
oc start-build myapp --from-dir=./src

# Build desde archivo tar
oc start-build myapp --from-file=app.tar.gz

# Build desde stdin
tar -cf - . | oc start-build myapp --from-archive=-
```

```yaml
# buildconfig-binary.yaml
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: binary-build
spec:
  source:
    type: Binary
    binary: {}
  strategy:
    type: Source
    sourceStrategy:
      from:
        kind: ImageStreamTag
        name: nodejs:latest
        namespace: openshift
  output:
    to:
      kind: ImageStreamTag
      name: myapp:latest
```

**4. Custom:**

```yaml
# BuildConfig con custom strategy
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: custom-build
spec:
  source:
    type: Git
    git:
      uri: https://github.com/user/myapp.git
  strategy:
    type: Custom
    customStrategy:
      from:
        kind: DockerImage
        name: my-builder:latest
      exposeDockerSocket: true
  output:
    to:
      kind: ImageStreamTag
      name: myapp:latest
```

**Gestión de builds:**

```bash
# Ver BuildConfigs
oc get bc

# Ver builds
oc get builds

# Iniciar build
oc start-build bc/myapp

# Iniciar con seguimiento
oc start-build bc/myapp --follow

# Cancelar build
oc cancel-build myapp-5

# Re-run un build
oc retry-build myapp-5

# Ver logs
oc logs build/myapp-5

# Ver historial
oc logs bc/myapp

# Limpiar builds antiguos
oc delete builds --field-selector status=Complete
```

---

### Integración con registros externos

**Configurar acceso a registro privado:**

```bash
# Crear secret con credenciales
oc create secret docker-registry quay-secret \
  --docker-server=quay.io \
  --docker-username=myuser \
  --docker-password=mypassword

# Asociar secret al ServiceAccount default
oc secrets link default quay-secret --for=pull

# Para builds
oc secrets link builder quay-secret
```

**Importar imágenes de registro externo:**

```yaml
# imagestream-external.yaml
apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  name: external-app
spec:
  tags:
  - name: latest
    from:
      kind: DockerImage
      name: quay.io/myuser/myapp:latest
    importPolicy:
      scheduled: true  # Importar cada 15 min
    referencePolicy:
      type: Source  # O Local para copiar al registry interno
```

**Registry interno de OpenShift:**

```bash
# Ver registry interno
oc get route -n openshift-image-registry

# Login al registry interno
oc registry login

# Push al registry interno
oc tag myapp:latest image-registry.openshift-image-registry.svc:5000/myproject/myapp:latest

# Exponer registry externamente
oc patch configs.imageregistry.operator.openshift.io/cluster \
  --type merge -p '{"spec":{"defaultRoute":true}}'

# URL del registry
oc get route default-route -n openshift-image-registry -o jsonpath='{.spec.host}'
```

---

## 3.3 Despliegue y exposición de aplicaciones

### oc new-app: flujo completo

`oc new-app` es el comando más potente para crear aplicaciones.

**Sintaxis:**
```bash
oc new-app [IMAGEN]~[CÓDIGO_FUENTE] [opciones]
```

**Ejemplos:**

```bash
# Desde imagen pública
oc new-app nginx:alpine --name=web

# Desde código fuente con S2I
oc new-app nodejs~https://github.com/user/nodejs-app.git

# Desde Dockerfile en repo
oc new-app https://github.com/user/docker-app.git --docker-image

# Con variables de entorno
oc new-app nginx:alpine \
  --name=web \
  -e ENVIRONMENT=production \
  -e LOG_LEVEL=info

# Con secrets
oc new-app postgresql \
  --name=db \
  --param=POSTGRESQL_USER=admin \
  --param=POSTGRESQL_PASSWORD=secret \
  --param=POSTGRESQL_DATABASE=mydb

# Múltiples componentes
oc new-app nodejs~https://github.com/user/frontend.git \
  postgresql \
  --name=myapp \
  --group frontend db

# Con labels
oc new-app nginx:alpine -l app=web,env=dev

# Dry-run (ver qué se crearía)
oc new-app nginx:alpine --dry-run -o yaml
```

**Recursos creados por new-app:**
- BuildConfig (si es S2I)
- DeploymentConfig o Deployment
- Service
- ImageStream

**Crear route automáticamente:**

```bash
# new-app + expose en un paso
oc new-app nodejs~https://github.com/user/app.git
oc expose svc/nodejs-app --hostname=app.apps.mycluster.com
```

---

### Routes: Edge, Passthrough y Re-encrypt TLS

Ya cubierto en el Día 2, pero aquí ejemplos adicionales:

**Route con certificado personalizado:**

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: secure-app
spec:
  host: secure.apps.cluster.example.com
  to:
    kind: Service
    name: webapp
  tls:
    termination: edge
    certificate: |
      -----BEGIN CERTIFICATE-----
      MIIFXTCCBEWgAwIBAgIJAJ...
      -----END CERTIFICATE-----
    key: |
      -----BEGIN RSA PRIVATE KEY-----
      MIIEpAIBAAKCAQEAw7...
      -----END RSA PRIVATE KEY-----
    caCertificate: |
      -----BEGIN CERTIFICATE-----
      MIIFXTCCBEWgAwIBAgI...
      -----END CERTIFICATE-----
```

**Route con path rewriting:**

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: api-route
  annotations:
    haproxy.router.openshift.io/rewrite-target: /
spec:
  host: api.apps.cluster.example.com
  path: /v1
  to:
    kind: Service
    name: api
```

**Rate limiting en Routes:**

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: limited-route
  annotations:
    haproxy.router.openshift.io/rate-limit-connections: "true"
    haproxy.router.openshift.io/rate-limit-connections.concurrent-tcp: "10"
    haproxy.router.openshift.io/rate-limit-connections.rate-http: "100"
    haproxy.router.openshift.io/rate-limit-connections.rate-tcp: "50"
spec:
  to:
    kind: Service
    name: webapp
```

---

### DeploymentConfig (legacy) vs Deployment

**Deployment:** Estándar de Kubernetes. Preferido.

**DeploymentConfig:** Específico de OpenShift (legacy). Ofrece:
- Hooks (pre/post deployment)
- Triggers personalizados
- Estrategias custom

```yaml
# DeploymentConfig (legacy)
apiVersion: apps.openshift.io/v1
kind: DeploymentConfig
metadata:
  name: webapp-dc
spec:
  replicas: 3
  selector:
    app: webapp
  strategy:
    type: Rolling
    rollingParams:
      updatePeriodSeconds: 1
      intervalSeconds: 1
      timeoutSeconds: 600
      pre:
        failurePolicy: Abort
        execNewPod:
          command: ["/bin/sh", "-c", "migrate.sh"]
          containerName: webapp
  triggers:
  - type: ConfigChange
  - type: ImageChange
    imageChangeParams:
      automatic: true
      containerNames:
      - webapp
      from:
        kind: ImageStreamTag
        name: webapp:latest
  template:
    spec:
      containers:
      - name: webapp
        image: webapp:latest
```

**Migrar de DeploymentConfig a Deployment:**

```bash
# Ver DeploymentConfigs existentes
oc get dc

# Crear Deployment desde DC
oc get dc webapp-dc -o yaml | \
  yq 'del(.metadata.resourceVersion, .metadata.uid, .metadata.creationTimestamp, .metadata.generation, .status, .spec.strategy.type, .spec.triggers)' | \
  sed 's/DeploymentConfig/Deployment/' | \
  sed 's/apps.openshift.io\/v1/apps\/v1/' | \
  oc apply -f -
```

---

### Estrategias: Rolling, Recreate, Blue/Green, Canary

**Rolling Update (por defecto):**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Pod extra durante update
      maxUnavailable: 1  # Pods que pueden estar down
```

**Recreate:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3
  strategy:
    type: Recreate  # Mata todos, luego crea todos
```

**Blue/Green:**

```yaml
# Deployment Blue (actual)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
      version: blue
  template:
    metadata:
      labels:
        app: webapp
        version: blue
    spec:
      containers:
      - name: web
        image: myapp:v1
---
# Deployment Green (nuevo)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
      version: green
  template:
    metadata:
      labels:
        app: webapp
        version: green
    spec:
      containers:
      - name: web
        image: myapp:v2
---
# Service apunta a Blue
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  selector:
    app: webapp
    version: blue  # Cambiar a 'green' para switch
  ports:
  - port: 80
```

```bash
# Switch a Green
oc patch service webapp -p '{"spec":{"selector":{"version":"green"}}}'

# Rollback a Blue
oc patch service webapp -p '{"spec":{"selector":{"version":"blue"}}}'
```

**Canary:**

```yaml
# Deployment stable (90% tráfico)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: webapp
      track: stable
  template:
    metadata:
      labels:
        app: webapp
        track: stable
    spec:
      containers:
      - name: web
        image: myapp:v1
---
# Deployment canary (10% tráfico)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webapp
      track: canary
  template:
    metadata:
      labels:
        app: webapp
        track: canary
    spec:
      containers:
      - name: web
        image: myapp:v2
---
# Service apunta a ambos
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  selector:
    app: webapp
  ports:
  - port: 80
```

```bash
# Incrementar canary
oc scale deployment webapp-canary --replicas=2
oc scale deployment webapp-stable --replicas=8

# Promover canary a stable
oc set image deployment/webapp-stable web=myapp:v2
oc scale deployment webapp-stable --replicas=10
oc scale deployment webapp-canary --replicas=0
```

---

## 3.4 Seguridad específica de OpenShift

### Security Context Constraints (SCC)

SCCs son policies de seguridad específicas de OpenShift que controlan:
- Permisos de contenedor (root, capabilities)
- SELinux context
- Usuarios/grupos
- Volúmenes permitidos

**SCCs predefinidos:**
- `restricted` - Restringido, no root
- `anyuid` - Permite cualquier UID
- `privileged` - Sin restricciones
- `hostaccess` - Acceso a host paths
- `hostmount-anyuid` - Host mounts con cualquier UID
- `node-exporter` - Para Node Exporter
- `nonroot` - Force non-root

```bash
# Ver SCCs disponibles
oc get scc

# Describir SCC
oc describe scc restricted
oc describe scc anyuid
oc describe scc privileged

# Ver qué SCC usa un Pod
oc get pod <pod-name> -o jsonpath='{.metadata.annotations.openshift\.io/scc}'
```

**Asignar SCC a ServiceAccount:**

```bash
# Crear ServiceAccount
oc create serviceaccount privileged-sa

# Otorgar SCC privileged
oc adm policy add-scc-to-user privileged -z privileged-sa

# Usar en Pod
oc set serviceaccount deployment/myapp privileged-sa
```

**Ejemplo - Contenedor que necesita root:**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: root-sa
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: use-anyuid
rules:
- apiGroups: ["security.openshift.io"]
  resources: ["securitycontextconstraints"]
  resourceNames: ["anyuid"]
  verbs: ["use"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: root-sa-anyuid
subjects:
- kind: ServiceAccount
  name: root-sa
roleRef:
  kind: Role
  name: use-anyuid
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: root-app
spec:
  selector:
    matchLabels:
      app: root-app
  template:
    metadata:
      labels:
        app: root-app
    spec:
      serviceAccountName: root-sa
      containers:
      - name: app
        image: nginx
        securityContext:
          runAsUser: 0  # root
```

**Custom SCC:**

```yaml
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: custom-scc
allowPrivilegedContainer: false
allowHostNetwork: false
allowHostPorts: false
allowHostDirVolumePlugin: false
allowHostPID: false
allowHostIPC: false
runAsUser:
  type: MustRunAsRange
  uidRangeMin: 1000
  uidRangeMax: 2000
seLinuxContext:
  type: MustRunAs
fsGroup:
  type: MustRunAs
  ranges:
  - min: 1000
    max: 2000
supplementalGroups:
  type: MustRunAs
  ranges:
  - min: 1000
    max: 2000
volumes:
- configMap
- downwardAPI
- emptyDir
- persistentVolumeClaim
- projected
- secret
```

---

### Relación SCC ↔ ServiceAccount

El scheduler de OpenShift selecciona el SCC más restrictivo que el Pod puede usar:

**Orden de selección:**
1. SCC especificado explícitamente (si tiene permiso)
2. SCCs disponibles para el ServiceAccount
3. SCC más restrictivo que satisface los requisitos del Pod

**Flujo de decisión:**

```
Pod creado
    ↓
Verificar ServiceAccount
    ↓
Verificar SCCs permitidos al SA
    ↓
Verificar requisitos del Pod (runAsUser, volumes, etc.)
    ↓
Seleccionar SCC más restrictivo compatible
    ↓
Aplicar contexto de seguridad
```

```bash
# Ver qué SCC puede usar un SA
oc describe scc | grep -A5 "Users:"

# Ver SCC asignado a un Pod
oc get pod <pod> -o jsonpath='{.metadata.annotations.openshift\.io/scc}'

# Debug de denegación SCC
oc adm policy who-can use scc privileged
```

---

### OAuth integrado: proveedores de identidad

OpenShift incluye OAuth server para autenticación.

**Proveedores soportados:**
- HTPasswd (archivo local)
- LDAP
- Keystone (OpenStack)
- GitHub / GitHub Enterprise
- GitLab
- Google
- OpenID Connect (OIDC)

**Configurar HTPasswd:**

```bash
# Crear archivo htpasswd
htpasswd -c users.htpasswd admin
htpasswd users.htpasswd developer

# Crear secret
oc create secret generic htpasswd-secret \
  --from-file=htpasswd=users.htpasswd \
  -n openshift-config

# Configurar OAuth
oc apply -f - <<EOF
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: htpasswd_provider
    mappingMethod: claim
    type: HTPasswd
    htpasswd:
      fileData:
        name: htpasswd-secret
EOF

# Asignar rol admin
oc adm policy add-cluster-role-to-user cluster-admin admin

# Login
oc login -u admin -p <password>
```

**Configurar GitHub OAuth:**

```yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: github
    mappingMethod: claim
    type: GitHub
    github:
      clientID: my-github-client-id
      clientSecret:
        name: github-client-secret
      organizations:
      - my-org
```

---

### Gestión de certificados del clúster

OpenShift gestiona certificados automáticamente, pero puedes personalizarlos.

**Certificados automáticos:**
- API server
- Router
- Service CA
- etcd
- Kubelet

```bash
# Ver certificados
oc get certificates -A

# Ver certificado del API server
oc get secret kube-apiserver-serving-ca \
  -n openshift-kube-apiserver -o jsonpath='{.data.ca\.crt}' | \
  base64 -d | openssl x509 -text -noout

# Ver certificados del router
oc get secret router-ca -n openshift-ingress-operator \
  -o jsonpath='{.data.tls\.crt}' | \
  base64 -d | openssl x509 -text -noout

# Rotar certificado del router
oc delete secret router-ca -n openshift-ingress-operator
# Se regenera automáticamente
```

**Certificado custom para API:**

```yaml
apiVersion: config.openshift.io/v1
kind: APIServer
metadata:
  name: cluster
spec:
  servingCerts:
    namedCertificates:
    - names:
      - api.mycluster.example.com
      servingCertificate:
        name: api-cert
```

**Certificado custom para Routes:**

```bash
# Crear secret TLS
oc create secret tls custom-route-cert \
  --cert=path/to/cert.crt \
  --key=path/to/cert.key

# Usar en Route
oc create route edge custom-route \
  --service=webapp \
  --hostname=custom.apps.mycluster.example.com \
  --cert=custom-route-cert
```

---

## Ejercicio práctico integrador - Día 3

Despliega una aplicación completa con S2I, seguridad y exposición:

```bash
# 1. Crear proyecto
oc new-project ejercicio-dia3 \
  --description="Ejercicio Día 3"

# 2. Crear aplicación con S2I desde GitHub
oc new-app nodejs~https://github.com/sclorg/nodejs-ex.git \
  --name=webapp \
  -e NODE_ENV=production

# 3. Ver build en progreso
oc logs -f bc/webapp

# 4. Exponer aplicación
oc expose svc/webapp --hostname=webapp.apps.cluster.example.com

# 5. Crear ServiceAccount con permisos especiales
oc create serviceaccount webapp-sa
oc adm policy add-scc-to-user anyuid -z webapp-sa

# 6. Actualizar deployment para usar SA
oc set serviceaccount deployment/webapp webapp-sa

# 7. Agregar ConfigMap
cat <<EOF | oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-config
data:
  LOG_LEVEL: "debug"
  CACHE_TTL: "3600"
EOF

oc set env deployment/webapp --from=configmap/webapp-config

# 8. Agregar Secret
oc create secret generic webapp-secret \
  --from-literal=API_KEY=secret-api-key-12345

oc set env deployment/webapp --from=secret/webapp-secret

# 9. Configurar Route con TLS
oc create route edge webapp-secure \
  --service=webapp \
  --hostname=secure.apps.cluster.example.com

# 10. Verificar
oc get all
oc get route
oc describe deployment webapp

# 11. Probar
curl -k https://webapp.apps.cluster.example.com
curl -k https://secure.apps.cluster.example.com

# 12. Limpiar
oc delete project ejercicio-dia3
```

---

## Resumen del Día 3

**Conceptos clave:**
- OKD = versión community, OCP = versión empresarial
- Router (HAProxy) = ingress controller de OpenShift
- Registry integrado = almacenamiento de imágenes interno
- ImageStreams = abstracción sobre imágenes con tracking de tags
- S2I = build de imágenes desde código sin Dockerfile
- BuildConfig = definición de builds (Source, Docker, Binary, Custom)
- oc new-app = comando mágico para crear aplicaciones completas
- Projects = Namespaces con aislamiento y quotas por defecto
- SCCs = control de seguridad para contenedores
- OAuth = autenticación integrada con múltiples proveedores
- Certificados gestionados automáticamente (API, Router, Services)

**Comandos esenciales del día:**
```bash
# OpenShift específicos
oc new-project / oc project
oc new-app
oc expose
oc start-build / oc logs bc/
oc import-image / oc tag
oc get is / oc get bc / oc get builds

# Seguridad
oc adm policy add-scc-to-user
oc adm policy add-role-to-user
oc describe scc

# Debug
oc debug pod/
oc debug node/
oc adm top pods / oc adm top nodes
```
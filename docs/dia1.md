# OpenShift Container Platform - Día 1
## Contenedores y la Plataforma de Orquestación

---

## 1.1 Introducción a los contenedores

### ¿Qué es un contenedor?

Un contenedor es una unidad ligera, portátil y auto-suficiente que empaqueta una aplicación con todas sus dependencias (bibliotecas, configuraciones, runtime). A diferencia de una máquina virtual, los contenedores **comparten el kernel del host**, lo que los hace mucho más eficientes.

**Diferencias clave: VM vs Contenedor**

| Característica | Máquina Virtual | Contenedor |
|----------------|-----------------|------------|
| Aislamiento | Hardware completo (hypervisor) | A nivel de proceso (kernel) |
| Tiempo de arranque | Minutos | Milisegundos/segundos |
| Tamaño | GB (SO completo) | MB (solo app + deps) |
| Densidad | Pocas por host | Cientos por host |
| Portabilidad | Limitada | Alta (OCI standard) |

**¿Por qué importa?** Como desarrollador, imagina que tu app funciona en tu laptop pero falla en producción. Los contenedores resuelven el clásico "en mi máquina funciona" porque empaquetan TODO lo necesario.

---

### Namespaces y cgroups: la magia del kernel Linux

Los contenedores no son magia, son características del kernel Linux combinadas:

**Namespaces (aislamiento):**
Cada contenedor ve su propia vista del sistema:
- `pid` - Propio árbol de procesos (PID 1 es el proceso principal del contenedor)
- `net` - Propia pila de red (interfaces, puertos, routing)
- `mnt` - Propio sistema de archivos
- `uts` - Propio hostname
- `ipc` - Propios recursos IPC (semáforos, memoria compartida)
- `user` - Propios usuarios/grupos (UID 0 dentro ≠ UID 0 fuera)

**cgroups (limitación de recursos):**
Controlan cuánto puede usar cada contenedor:
- CPU (shares, cores, quota)
- Memoria (límites hard/soft)
- I/O de disco
- Red (bandwidth)

**Ejemplo práctico - Ver namespaces de un proceso:**

```bash
# Ejecuta un contenedor simple
podman run -d --name demo-ns alpine sleep 3600

# Encuentra el PID del proceso principal
PID=$(podman inspect demo-ns --format '{{.State.Pid}}')
echo "PID del contenedor: $PID"

# Mira sus namespaces
ls -la /proc/$PID/ns/
# Verás que cada namespace es un enlace simbólico diferente

# Compara con tu shell actual
ls -la /proc/self/ns/

# ¡Son diferentes! Ese es el aislamiento
```

**Ejemplo - Límites con cgroups:**

```bash
# Contenedor con límites de memoria y CPU
podman run -d --name limited \
  --memory 256m \
  --cpus 0.5 \
  alpine sleep 3600

# Verifica los límites aplicados
cat /sys/fs/cgroup/memory.slice/memory.limit_in_bytes 2>/dev/null || \
  cat /sys/fs/cgroup/memory.max 2>/dev/null

# Intenta usar más memoria de la permitida (fallará)
podman exec limited sh -c "dd if=/dev/zero bs=1M count=300 | md5sum"
# El OOM killer terminará el proceso
```

---

### Imágenes OCI: capas, registros y etiquetas

**¿Qué es una imagen OCI?**
Es un template de solo lectura para crear contenedores. Sigue el estándar Open Container Initiative (OCI), garantizando portabilidad entre runtimes (Podman, Docker, containerd, CRI-O).

**Sistema de capas (Union Filesystem):**
Las imágenes son capas apiladas de solo lectura:
```
┌─────────────────────────┐ ← Capa writable (contenedor)
├─────────────────────────┤ ← Capa 4: configuración app
├─────────────────────────┤ ← Capa 3: dependencias npm
├─────────────────────────┤ ← Capa 2: Node.js runtime
├─────────────────────────┤ ← Capa 1: Sistema base (Ubuntu/Alpine)
└─────────────────────────┘
```

**Ventajas:**
- Capas compartidas entre imágenes → ahorro de espacio
- Builds más rápidos (solo cambian capas superiores)
- Downloads incrementales

**Ejemplo - Inspeccionar capas:**

```bash
# Descarga una imagen
podman pull node:18-alpine

# Mira sus capas
podman inspect node:18-alpine --format '{{json .RootFS.Layers}}' | jq

# Historial de la imagen (qué añadió cada capa)
podman history node:18-alpine

# Compara tamaños de imágenes base
podman pull alpine:3.18
podman pull ubuntu:22.04
podman images | grep -E "alpine|ubuntu"
# Alpine ~5MB vs Ubuntu ~77MB
```

**Registros de imágenes:**
Donde se almacenan las imágenes:
- Docker Hub (hub.docker.com) - público por defecto
- Quay.io - de Red Hat, ideal para OpenShift
- GitHub Container Registry (ghcr.io)
- Registros privados (OpenShift tiene el suyo integrado)

**Etiquetas (tags):**
Versionado de imágenes:
- `node:18` - tag específico
- `node:18-alpine` - tag con variante
- `node:latest` - tag mutable (¡evítalo en producción!)

**Ejemplo - Trabajo con registros:**

```bash
# Login en un registro
podman login quay.io

# Etiquetar imagen para push
podman tag node:18-alpine quay.io/mi-usuario/mi-app:v1.0

# Subir imagen
podman push quay.io/mi-usuario/mi-app:v1.0

# Ver digest (hash único, inmutable)
podman inspect quay.io/mi-usuario/mi-app:v1.0 --format '{{.Digest}}'
# sha256:abc123... ← Útil para pines de seguridad
```

---

### Gestión de contenedores con Podman

Podman (Pod Manager) es el runtime de contenedores preferido en el ecosistema Red Hat/OpenShift. Es daemonless y rootless por diseño.

**Comandos esenciales:**

```bash
# === CICLO DE VIDA ===

# Ejecutar contenedor interactivo
podman run -it alpine sh

# Ejecutar en background (detached)
podman run -d --name web -p 8080:80 nginx:alpine

# Listar contenedores activos
podman ps

# Listar todos (incluidos parados)
podman ps -a

# Detener contenedor
podman stop web

# Iniciar contenedor existente
podman start web

# Reiniciar
podman restart web

# Eliminar contenedor (debe estar parado)
podman rm web

# Forzar eliminación
podman rm -f web

# === EJECUCIÓN EN CONTENEDORES ===

# Ejecutar comando en contenedor activo
podman exec -it web sh

# Copiar archivos
podman cp ./archivo.txt web:/tmp/

# Ver logs
podman logs web
podman logs -f web  # follow (streaming)

# === IMÁGENES ===

# Listar imágenes locales
podman images

# Eliminar imagen
podman rmi nginx:alpine

# Eliminar imágenes no usadas
podman image prune

# Construir imagen desde Dockerfile
podman build -t mi-app:v1.0 .

# === INSPECCIÓN ===

# Detalles del contenedor
podman inspect web

# Uso de recursos en tiempo real
podman stats

# Procesos dentro del contenedor
podman top web
```

**Ejemplo completo - Aplicación web simple:**

```bash
# 1. Crear Dockerfile
mkdir demo-web && cd demo-web
cat > Dockerfile << 'EOF'
FROM python:3.11-alpine
WORKDIR /app
COPY server.py .
EXPOSE 8000
CMD ["python", "server.py"]
EOF

# 2. Crear aplicación
cat > server.py << 'EOF'
from http.server import HTTPServer, SimpleHTTPRequestHandler
import os

class Handler(SimpleHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header('Content-type', 'text/plain')
        self.end_headers()
        hostname = os.uname().nodename
        self.wfile.write(f"¡Hola desde {hostname}!\n".encode())

if __name__ == '__main__':
    server = HTTPServer(('0.0.0.0', 8000), Handler)
    print("Servidor iniciado en puerto 8000")
    server.serve_forever()
EOF

# 3. Construir imagen
podman build -t demo-web:v1.0 .

# 4. Ejecutar
podman run -d --name web-demo -p 8000:8000 demo-web:v1.0

# 5. Probar
curl http://localhost:8000

# 6. Ver logs
podman logs web-demo

# 7. Limpiar
podman rm -f web-demo
```

**Podman vs Docker:**
| Aspecto | Docker | Podman |
|---------|--------|--------|
| Arquitectura | Daemon (dockerd) | Daemonless |
| Permisos | Requiere root o rootless setup | Rootless nativo |
| Pods | No (solo contenedores) | Sí (compatibilidad Kubernetes) |
| Systemd | Integración manual | Integración nativa (`podman generate systemd`) |
| Seguridad | Contexto SELinux opcional | SELinux por defecto |

---

## 1.2 Arquitectura del clúster OpenShift

OpenShift es una plataforma de contenedores empresarial basada en Kubernetes. Añade capacidades de PaaS, seguridad reforzada y herramientas para desarrolladores.

### Componentes del Control Plane (Master Node)

El cerebro del clúster:

**API Server (`openshift-apiserver`)**
- Punto de entrada único para todas las llamadas REST
- Autentica y autoriza requests
- Valida objetos y escribe en etcd
- Puerto: 6443 (HTTPS)

**etcd**
- Base de datos clave-valor distribuida
- Almacena TODO el estado del clúster
- Requiere backups regulares
- Puerto: 2379

**Scheduler (`openshift-kube-scheduler`)**
- Decide dónde colocar nuevos Pods
- Considera: recursos, afinidades, taints, policies
- No reasigna Pods existentes

**Controller Manager (`openshift-kube-controller-manager`)**
- Ejecuta bucles de control (control loops)
- Mantiene el estado deseado
- Ejemplos: Node Controller, Replication Controller, Service Account Controller

**OpenShift Controllers adicionales:**
- `openshift-controller-manager` - Controllers específicos de OpenShift
- `openshift-route-controller` - Gestión de Routes
- `openshift-image-registry` - Registro interno de imágenes

**Ejemplo - Flujo de creación de un Pod:**

```
Usuario (oc apply) 
    → API Server (autenticación, validación, escritura en etcd)
    → Scheduler (detecta Pod sin nodo, calcula mejor nodo)
    → API Server (actualiza Pod con nodo asignado)
    → Kubelet en nodo asignado (detecta nuevo Pod, crea contenedores via CRI)
    → Pod ejecutándose
```

---

### Componentes del Nodo (Worker Node)

**kubelet**
- Agente que corre en cada nodo
- Se comunica con API Server
- Gestiona ciclo de vida de Pods (crea, arranca, monitoriza, destruye)
- Reporta estado del nodo

**kube-proxy**
- Mantiene reglas de red (iptables/nftables o IPVS)
- Implementa abstracción Service
- Balancea tráfico a Pods backend

**CRI (Container Runtime Interface)**
- OpenShift usa **CRI-O** (no Docker, no containerd)
- Ligero, diseñado para Kubernetes
- Compatible OCI

**Otros componentes en el nodo:**
- **OVN-Kubernetes** - CNI plugin para networking
- **Machine Config Daemon** - Aplica configuraciones del nodo
- **Prometheus Node Exporter** - Métricas

**Ejemplo - Ver componentes en un clúster OpenShift:**

```bash
# Si tienes acceso a un clúster OpenShift:

# Login como admin
oc login -u kubeadmin -p <password> https://api.cluster.example.com:6443

# Ver nodos
oc get nodes
oc get nodes -o wide  # con más info

# Ver pods del control plane (corren en master nodes)
oc get pods -n openshift-kube-apiserver
oc get pods -n openshift-etcd
oc get pods -n openshift-kube-scheduler
oc get pods -n openshift-kube-controller-manager

# Ver pods de sistema en un nodo worker
oc get pods -n openshift-ovn-kubernetes -o wide
oc get pods -n openshift-machine-config-operator

# Inspeccionar un nodo
oc describe node <nombre-nodo>

# Ver versiones de componentes
oc get clusterversion
oc describe clusterversion
```

---

## 1.3 Recursos básicos del clúster

### Pods: ciclo de vida y estados

Un Pod es la unidad mínima desplegable en Kubernetes/OpenShift. Puede contener uno o más contenedores que comparten:
- Dirección IP
- Volúmenes
- Espacio de nombres de red (localhost entre contenedores del mismo Pod)

**Estados del Pod:**
- `Pending` - Aceptado pero no iniciado (esperando recursos)
- `Running` - Al menos un contenedor ejecutándose
- `Succeeded` - Todos los contenedores terminaron con éxito
- `Failed` - Al menos un contenedor terminó con error
- `Unknown` - Estado no determinado (problema de comunicación)

**Ciclo de vida detallado:**

```
Creation → Pending → Running → [Succeeded | Failed]
                    ↓
              CrashLoopBackOff (restart policy + fallos repetidos)
```

**Conditions (condiciones del Pod):**
- `PodScheduled` - Asignado a un nodo
- `Ready` - Listo para recibir tráfico
- `Initialized` - Init containers completados
- `ContainersReady` - Todos los contenedores listos

**Ejemplo - Pod simple:**

```yaml
# pod-simple.yaml
apiVersion: v1
kind: Pod
metadata:
  name: mi-pod
  labels:
    app: demo
spec:
  containers:
  - name: web
    image: nginx:alpine
    ports:
    - containerPort: 80
    resources:
      requests:
        memory: "64Mi"
        cpu: "50m"
      limits:
        memory: "128Mi"
        cpu: "100m"
```

```bash
# Crear el Pod
oc apply -f pod-simple.yaml

# Ver estado
oc get pods mi-pod
oc get pod mi-pod -o wide

# Ver detalles y eventos
oc describe pod mi-pod

# Ver logs
oc logs mi-pod

# Ejecutar comando dentro del Pod
oc exec mi-pod -- curl localhost

# Ver YAML del Pod en el clúster
oc get pod mi-pod -o yaml

# Eliminar
oc delete pod mi-pod
```

**Ejemplo - Pod con múltiples contenedores (sidecar):**

```yaml
# pod-multi.yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container
spec:
  containers:
  - name: app
    image: nginx:alpine
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/nginx
  - name: sidecar
    image: busybox
    command: ["sh", "-c", "tail -f /logs/access.log"]
    volumeMounts:
    - name: shared-logs
      mountPath: /logs
  volumes:
  - name: shared-logs
    emptyDir: {}
```

```bash
oc apply -f pod-multi.yaml
oc logs multi-container -c app      # logs del contenedor app
oc logs multi-container -c sidecar  # logs del sidecar
oc exec multi-container -c app -- ls /var/log/nginx
```

---

### ReplicaSets y Deployments

**ReplicaSet:** Mantiene un número estable de réplicas de Pods idénticos.

```yaml
# replicaset.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx:alpine
```

**Deployment:** Abstracción sobre ReplicaSet que añade:
- Actualizaciones declarativas
- Rollouts y rollbacks
- Historial de revisiones

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # puede crear 1 pod extra durante update
      maxUnavailable: 0  # nunca menos de 3 pods activos
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: web
        image: nginx:1.24-alpine
        ports:
        - containerPort: 80
```

```bash
# Crear deployment
oc apply -f deployment.yaml

# Ver estado
oc get deployments
oc get pods -l app=webapp

# Escalar
oc scale deployment webapp --replicas=5
oc get pods -l app=webapp

# Actualizar imagen (trigger rollout)
oc set image deployment/webapp web=nginx:1.25-alpine

# Ver progreso del rollout
oc rollout status deployment/webapp

# Ver historial
oc rollout history deployment/webapp

# Rollback
oc rollout undo deployment/webapp
oc rollout undo deployment/webapp --to-revision=1

# Ver revisiones
oc rollout history deployment/webapp --revision=2
```

---

### Services: ClusterIP, NodePort, LoadBalancer

Un Service expone un conjunto de Pods como servicio de red. Usa labels para encontrar los Pods backend.

**Tipos de Service:**

**ClusterIP (por defecto):**
- IP interna del clúster
- Solo accesible desde dentro del clúster
- Ideal para comunicación entre microservicios

**NodePort:**
- Expone un puerto en cada nodo (rango 30000-32767)
- Accesible desde fuera: `<NodeIP>:<NodePort>`
- Útil para desarrollo/testing

**LoadBalancer:**
- Provisiona un balanceador externo (AWS ELB, GCP LB, etc.)
- Requiere infraestructura cloud
- En OpenShift, usar Routes para HTTP/HTTPS

**Ejemplo completo:**

```yaml
# service-demo.yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-clusterip
spec:
  type: ClusterIP
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: webapp-nodeport
spec:
  type: NodePort
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

```bash
# Crear services
oc apply -f service-demo.yaml

# Ver services
oc get svc
oc get endpoints webapp-clusterip  # Pods backend

# Probar ClusterIP desde dentro del clúster
oc run test --rm -it --image=busybox -- wget -qO- http://webapp-clusterip

# Probar NodePort
oc get nodes -o wide  # obtener IP de nodo
curl http://<NODE_IP>:30080

# En OpenShift, preferimos Routes para exposición HTTP
oc expose service webapp-clusterip --hostname=webapp.apps.cluster.example.com
```

---

### Proyectos y organización de recursos en OpenShift

Un **Proyecto** en OpenShift es un Namespace de Kubernetes con características adicionales:
- Aislamiento de red por defecto
- Quotas y limits preconfigurados
- Roles y permisos específicos

```bash
# Listar proyectos
oc get projects

# Crear proyecto
oc new-project mi-app --description="Proyecto de demostración"

# Cambiar de proyecto
oc project mi-app

# Ver proyecto actual
oc project

# Ver todos los recursos en un proyecto
oc get all

# Describir proyecto
oc describe project mi-app

# Eliminar proyecto
oc delete project mi-app
```

---

## 1.4 Configuración de aplicaciones

### ConfigMaps

Los ConfigMaps desacoplan la configuración de las imágenes de contenedor. Permiten cambiar configuración sin reconstruir la imagen.

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  # Valores clave-valor simples
  database_host: "postgres.mi-namespace.svc.cluster.local"
  database_port: "5432"
  log_level: "info"
  
  # Archivo completo
  app.properties: |
    cache.enabled=true
    cache.ttl=3600
    max.connections=100
```

**Uso como variables de entorno:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-demo
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "env | grep DB && sleep 3600"]
    env:
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: database_host
    - name: DB_PORT
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: database_port
```

**Uso como volumen:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-volume-demo
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "cat /etc/config/app.properties && sleep 3600"]
    volumeMounts:
    - name: config-vol
      mountPath: /etc/config
  volumes:
  - name: config-vol
    configMap:
      name: app-config
```

```bash
# Crear ConfigMap desde literal
oc create configmap db-config --from-literal=host=localhost --from-literal=port=5432

# Crear desde archivo
oc create configmap app-config --from-file=app.properties

# Crear desde directorio
oc create configmap full-config --from-file=./config/

# Ver ConfigMap
oc get configmap
oc describe configmap app-config

# Editar
oc edit configmap app-config
```

---

### Secrets

Los Secrets almacenan datos sensibles (contraseñas, tokens, claves). Se codifican en base64 y pueden encriptarse en reposo.

**Tipos de Secrets:**
- `Opaque` - Genérico (por defecto)
- `kubernetes.io/basic-auth` - Credenciales
- `kubernetes.io/ssh-auth` - Claves SSH
- `kubernetes.io/tls` - Certificados TLS
- `kubernetes.io/dockerconfigjson` - Credenciales de registro

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:  # stringData es más fácil (no requiere base64 manual)
  username: admin
  password: supersecretpassword123
```

```bash
# Crear secret desde literales
oc create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=secreto123

# Crear secret TLS
oc create secret tls mi-certificado \
  --cert=path/to/cert.crt \
  --key=path/to/cert.key

# Crear secret para registro de imágenes
oc create secret docker-registry mi-registry \
  --docker-server=quay.io \
  --docker-username=miuser \
  --docker-password=mipassword

# Ver secret (solo metadata)
oc get secrets
oc describe secret db-credentials

# Decodificar valor
oc get secret db-credentials -o jsonpath='{.data.password}' | base64 -d
```

**Uso en Pods:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-demo
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo $DB_USER:$DB_PASS && sleep 3600"]
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DB_PASS
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
```

---

## 1.5 Almacenamiento

### Volumes efímeros

**emptyDir:** Volumen temporal que vive mientras el Pod existe. Útil para caché, datos temporales.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-demo
spec:
  containers:
  - name: writer
    image: busybox
    command: ["sh", "-c", "echo 'Hola' > /data/mensaje && sleep 3600"]
    volumeMounts:
    - name: temp-data
      mountPath: /data
  - name: reader
    image: busybox
    command: ["sh", "-c", "cat /data/mensaje && sleep 3600"]
    volumeMounts:
    - name: temp-data
      mountPath: /data
  volumes:
  - name: temp-data
    emptyDir: {}
```

**ConfigMap y Secret como volúmenes:**
Ya vistos arriba. Son efímeros en el sentido de que se actualizan cuando el recurso cambia.

---

### PersistentVolumes y PersistentVolumeClaims

**PersistentVolume (PV):** Recurso de almacenamiento a nivel de clúster.
**PersistentVolumeClaim (PVC):** Solicitud de almacenamiento por un usuario/proyecto.

Flujo:
```
Desarrollador → PVC → (binding) → PV → (backend) → Almacenamiento físico
```

**Modos de acceso:**
- `ReadWriteOnce (RWO)` - Un nodo puede montar como RW
- `ReadOnlyMany (ROX)` - Múltiples nodos pueden montar como RO
- `ReadWriteMany (RWX)` - Múltiples nodos pueden montar como RW
- `ReadWriteOncePod (RWOP)` - Un solo Pod puede montar como RW

```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-storage
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: standard  # depende del clúster
```

```yaml
# pod-con-pvc.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-con-storage
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /usr/share/nginx/html
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: data-storage
```

```bash
# Crear PVC
oc apply -f pvc.yaml

# Ver estado
oc get pvc
oc describe pvc data-storage

# Ver PVs (a nivel de clúster)
oc get pv

# Usar en un Pod
oc apply -f pod-con-pvc.yaml

# Escribir datos persistentes
oc exec app-con-storage -- sh -c "echo 'Datos persistentes' > /usr/share/nginx/html/index.html"

# Eliminar y recrear Pod
oc delete pod app-con-storage
oc apply -f pod-con-pvc.yaml

# Verificar que datos persisten
oc exec app-con-storage -- cat /usr/share/nginx/html/index.html
```

---

### StorageClasses y aprovisionamiento dinámico

Una StorageClass define tipos de almacenamiento disponibles. Permite aprovisionamiento dinámico: crear PV automáticamente cuando se solicita un PVC.

```yaml
# storageclass.yaml (ejemplo - normalmente ya existen en el clúster)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: kubernetes.io/aws-ebs  # ejemplo AWS
parameters:
  type: gp3
reclaimPolicy: Delete  # Delete o Retain
volumeBindingMode: WaitForFirstConsumer
```

```bash
# Ver StorageClasses disponibles
oc get storageclass
oc describe storageclass

# Crear PVC con StorageClass específica
cat <<EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fast-data
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: fast-storage
  resources:
    requests:
      storage: 5Gi
EOF

# El PV se crea automáticamente cuando el PVC se usa
oc get pvc fast-data -w
```

**Reclaim Policies:**
- `Delete` - Elimina PV y datos cuando se borra el PVC
- `Retain` - Mantiene PV y datos (requiere cleanup manual)
- `Recycle` - (deprecado) Ejecuta `rm -rf` en el volumen

---

## Ejercicio práctico integrador - Día 1

Despliega una aplicación web completa con configuración y almacenamiento persistente:

```yaml
# ejercicio-dia1.yaml
# Configuración
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-config
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
---
apiVersion: v1
kind: Secret
metadata:
  name: webapp-secret
stringData:
  API_KEY: "mi-api-key-secreta"
---
# Almacenamiento
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: webapp-data
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
---
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: web
        image: nginx:alpine
        ports:
        - containerPort: 80
        envFrom:
        - configMapRef:
            name: webapp-config
        - secretRef:
            name: webapp-secret
        volumeMounts:
        - name: data
          mountPath: /usr/share/nginx/html
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: webapp-data
---
# Service
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
# Desplegar todo
oc apply -f ejercicio-dia1.yaml

# Verificar
oc get all,pvc,configmap,secret

# Probar
oc expose service webapp
oc get route

# Escalar
oc scale deployment webapp --replicas=3

# Limpiar
oc delete all,configmap,secret,pvc -l app=webapp
oc delete project <proyecto>  # si creaste uno nuevo
```

---

## Resumen del Día 1

**Conceptos clave:**
- Contenedores = Namespaces + cgroups (kernel Linux)
- Imágenes OCI = capas inmutables + registros + tags
- Podman = runtime daemonless, rootless, compatible con Docker
- OpenShift = Kubernetes + PaaS + Seguridad empresarial
- Pods = unidad mínima, efímeros, 1+ contenedores
- Deployments = ReplicaSets + estrategia de actualización
- Services = abstracción de red para acceso a Pods
- ConfigMaps/Secrets = configuración externalizada
- PVCs = almacenamiento persistente desacoplado del ciclo de vida del Pod

**Comandos esenciales del día:**
```bash
# Podman
podman run/build/images/ps/rm/rmi/exec/logs

# OpenShift
oc login/logout
oc get/describe/apply/delete/edit
oc project/new-project
oc logs/exec
oc scale/rollout
oc expose
```
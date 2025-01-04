# Manual para Instalar Kubernetes con K3s, Implementar un Cluster de Tres Nodos, Crear un Docker Registry en el Cluster y Desplegar una API

## Requisitos Previos
Antes de comenzar, asegúrate de cumplir con los siguientes requisitos:

- Tres máquinas (físicas o virtuales) con un sistema operativo basado en Linux (preferiblemente Ubuntu 20.04 o superior).
- Acceso root o privilegios de administrador en las máquinas.
- Conexión a Internet.
- Configuración básica de red entre los nodos.

## Paso 1: Instalar K3s

### 1.1 Instalar K3s en el Nodo Maestro
En el nodo maestro, instala K3s ejecutando el siguiente comando:
```bash
curl -sfL https://get.k3s.io | sh -s - server --write-kubeconfig-mode 644
printf "[DEBUG] Nodo maestro K3s instalado.\n"
```
Guarda el token para que los nodos worker puedan unirse al cluster:
```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

### 1.2 Instalar K3s en los Nodos Worker
En cada nodo worker, usa el siguiente comando para instalar K3s y unirte al cluster. Sustituye `<master-ip>` por la dirección IP del nodo maestro y `<token>` por el token obtenido en el paso anterior:
```bash
curl -sfL https://get.k3s.io | K3S_URL=https://<master-ip>:6443 K3S_TOKEN=<token> sh -
printf "[DEBUG] Nodo worker K3s instalado y unido al cluster.\n"
```

### 1.3 Verificar el Cluster
En el nodo maestro, verifica que todos los nodos estén activos:
```bash
kubectl get nodes
printf "[DEBUG] Estado del cluster verificado.\n"
```

## Paso 2: Configurar un Docker Registry en el Cluster

### 2.1 Crear el Deployment del Docker Registry
Crea un archivo `registry-deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: registry
  labels:
    app: registry
spec:
  replicas: 1
  selector:
    matchLabels:
      app: registry
  template:
    metadata:
      labels:
        app: registry
    spec:
      containers:
      - name: registry
        image: registry:2
        ports:
        - containerPort: 5000
```
Aplica el Deployment:
```bash
kubectl apply -f registry-deployment.yaml
printf "[DEBUG] Deployment del Docker Registry aplicado.\n"
```

### 2.2 Crear el Servicio del Docker Registry
Crea un archivo `registry-service.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: registry
spec:
  selector:
    app: registry
  ports:
    - protocol: TCP
      port: 5000
      targetPort: 5000
  type: NodePort
```
Aplica el Servicio:
```bash
kubectl apply -f registry-service.yaml
printf "[DEBUG] Servicio del Docker Registry aplicado y expuesto.\n"
```

### 2.3 Configurar el Cluster para Usar el Registry
Encuentra el puerto expuesto del Docker Registry:
```bash
kubectl get service registry
```
Anota el puerto asignado por el NodePort. Asegúrate de que los nodos puedan acceder al Docker Registry utilizando la IP del nodo maestro y el puerto expuesto, por ejemplo: `<master-ip>:<node-port>`.

### 2.4 Configurar los Equipos Clientes para Usar el Registry
En los equipos que utilizarán el Docker Registry, es necesario agregar su dirección al archivo de configuración de Docker como un registry inseguro. Edita el archivo `/etc/docker/daemon.json` o créalo si no existe, y añade lo siguiente:
```json
{
  "insecure-registries": ["<master-ip>:<node-port>"]
}
```
Reinicia el servicio Docker para aplicar los cambios:
```bash
sudo systemctl restart docker
printf "[DEBUG] Docker configurado para usar el registry del cluster.\n"
```

## Paso 3: Desplegar una API en el Cluster

### 3.1 Crear y Subir la Imagen de la API al Registry
En tu máquina local o en uno de los nodos con acceso al cluster, sube la imagen al registry del cluster. Sustituye `<master-ip>` y `<node-port>` con la dirección IP del nodo maestro y el puerto asignado al servicio del registry:
```bash
docker build -t <master-ip>:<node-port>/api:latest .
docker push <master-ip>:<node-port>/api:latest
printf "[DEBUG] Imagen de la API subida al registry del cluster.\n"
```

### 3.2 Crear un Deployment en K3s
Crea un archivo `api-deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: <master-ip>:<node-port>/api:latest
        ports:
        - containerPort: 80
```
Aplica el Deployment:
```bash
kubectl apply -f api-deployment.yaml
printf "[DEBUG] Deployment de la API aplicado.\n"
```

### 3.3 Exponer el Servicio
Crea un archivo `api-service.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api
  ports:
      - protocol: TCP
        port: 80
        targetPort: 80
  type: NodePort
```
Aplica el Servicio:
```bash
kubectl apply -f api-service.yaml
printf "[DEBUG] Servicio de la API expuesto.\n"
```

Obtén el puerto asignado:
```bash
kubectl get service api-service
printf "[DEBUG] Información del servicio obtenida.\n"
```
Accede a la API en `<node-ip>:<node-port>`.

## Conclusión
Con esto, has configurado un cluster K3s, implementado un Docker Registry dentro del cluster con acceso desde máquinas externas, configurado los equipos para usar el registry y desplegado una API utilizando dicho registry. ¡Listo para explorar más funcionalidades de K3s!

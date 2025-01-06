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
```
Guarda el token para que los nodos worker puedan unirse al cluster:
```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

### 1.2 Instalar K3s en los Nodos Worker
En cada nodo worker, usa el siguiente comando para instalar K3s y unirte al cluster. Sustituye `<master-ip>` por la dirección IP del nodo maestro y `<token>` por el token obtenido en el paso anterior:
```bash
curl -sfL https://get.k3s.io | K3S_URL=https://<master-ip>:6443 K3S_TOKEN=<token> sh -
```

### 1.3 Verificar el Cluster
En el nodo maestro, verifica que todos los nodos estén activos:
```bash
kubectl get nodes
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
```

### 2.2 Crear el Servicio del Docker Registry
Aplica el Despliegue y servicio de registry:
```bash
kubectl apply -f registry-deployment.yaml
```

### 2.3 Configurar el Cluster para Usar el Registry
Encuentra el puerto expuesto del Docker Registry:
```bash
kubectl get service registry
```
Anota el puerto asignado por el NodePort. Asegúrate de que los nodos puedan acceder al Docker Registry utilizando la IP del nodo maestro y el puerto expuesto, por ejemplo: `<master-ip>:<node-port>`.

### 2.4 Configurar registry en el cluster
Crea el archivo registries.yaml
```bash
sudo nano /etc/rancher/k3s/registries.yaml
```
```yaml
mirrors:
  registry.example.com:
    endpoint:
      - "http://url.example:5000"
```

### 2.5 Configurar los Equipos Clientes para Usar el Registry
En los equipos que utilizarán el Docker Registry, es necesario agregar su dirección al archivo de configuración de Docker como un registry inseguro. Edita el archivo `/etc/docker/daemon.json` o créalo si no existe, y añade lo siguiente:
```json
{
  "insecure-registries": ["<master-ip>:<node-port>"]
}
```
Reinicia el servicio Docker para aplicar los cambios:
```bash
sudo systemctl restart docker
```

## Paso 3: Desplegar una API en el Cluster

### 3.1 Crear y Subir la Imagen de la API al Registry
En tu máquina local o en uno de los nodos con acceso al cluster, sube la imagen al registry del cluster. Sustituye `<master-ip>` y `<node-port>` con la dirección IP del nodo maestro y el puerto asignado al servicio del registry:
```bash
docker build -t <master-ip>:<node-port>/<service-name>:latest .
docker push <master-ip>:<node-port>/<service-name>:latest
```

### 3.2 Desplegar servicio
Aplica el Deployment:
```bash
kubectl apply -f service-deployment.yaml
```

Obtén el puerto asignado:
```bash
kubectl get service service-name
```
Accede a la API en `<node-ip>:<node-port>`.
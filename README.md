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
### 1.2.1 Error en instalación de nodo
Es posible que en la instalación se encuentre un error similar a:
```bash
[E0109 20:32:07.393678 1776178 memcache.go:265] "Unhandled Error" err="couldn't get current server API group list: Get \"http://localhost:8080/api?timeout=32s\": dial tcp 127.0.0.1:8080: connect: connection refused"
```
Para esto será necesario copiar el archivo de /etc/rancher/k3s/k3s.yaml del nodo maestro en el nodo worker en la misma ruta remplazando **server** con la ip del nodo maestro y exponer el archivo de configuración con el siguiente comando:
```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

### 1.3 Verificar el Cluster
En el nodo maestro, verifica que todos los nodos estén activos:
```bash
kubectl get nodes
```

## Paso 2: Configurar un Docker Registry en el Cluster
Aplica el Deployment:
```bash
kubectl apply -f registry.yaml
```

### 2.3 Configurar el Cluster para Usar el Registry
Encuentra el puerto expuesto del Docker Registry:
```bash
kubectl get service registry
```
Asegúrate de que los nodos puedan acceder al Docker Registry utilizando la IP del nodo maestro y el puerto expuesto, por ejemplo: `<master-ip>:32000`.

### 2.4 Configurar registry en el cluster
Crea el archivo registries.yaml:
```bash
sudo nano /etc/rancher/k3s/registries.yaml
```
```yaml
mirrors:
  registry.name.com: #Remplaza por el nombre que le vayas a poner al registry
    endpoint:
      - "http://<master-ip>:32000"
```

Reinicia el servicio K3s en los nodos:
```bash
sudo systemctl restart k3s
```

### 2.5 Configurar los Equipos Clientes para Usar el Registry
En los equipos que utilizarán el Docker Registry, es necesario agregar su dirección al archivo de configuración de Docker como un registry inseguro. Edita el archivo `/etc/docker/daemon.json` o créalo si no existe, y añade lo siguiente:
```json
{
  "insecure-registries": ["registry.name.com:32000"]
}
```
Reinicia el servicio Docker para aplicar los cambios:
```bash
sudo systemctl restart docker
```
Añade la IP del registry a los hosts del equipo para poder nombrar las imagenes por el nombre del registry:
```bash
sudo nano /etc/hosts
```
Añade la linea:
```bash
10.43.34.135    registry.name.com
```

## Paso 4: Desplegar una API en el Cluster

### 4.1 Crear y Subir la Imagen de la API al Registry
En tu máquina local o en uno de los nodos con acceso al cluster, sube la imagen al registry del cluster. Sustituye `<master-ip>` y `<node-port>` con la dirección IP del nodo maestro y el puerto asignado al servicio del registry:
```bash
docker build -t registry.name.com:32000/<service-name>:latest .
docker push registry.name.com:32000/<service-name>:latest
```

### 4.2 Desplegar servicio
Aplica el Deployment:
```bash
kubectl apply -f service-deployment.yaml
```

Obtén el puerto asignado:
```bash
kubectl get service service-name
```
Accede a la API en `<node-ip>:<service-port>`.

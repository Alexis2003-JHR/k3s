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
Crea un archivo `registry-service.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: registry
  labels:
    app: registry
spec:
  type: NodePort
  ports:
    - port: 5000
      targetPort: 5000
      nodePort: 32000
  selector:
    app: registry
```
Aplica el Servicio:
```bash
kubectl apply -f registry-service.yaml
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
  "<master-ip>:32000":
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
  "insecure-registries": ["<master-ip>:32000"]
}
```
Reinicia el servicio Docker para aplicar los cambios:
```bash
sudo systemctl restart docker
```

## Paso 3: Instalar y Configurar Prometheus

### 3.1 Desplegar Prometheus en el Cluster
Crea un archivo `prometheus-deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  labels:
    app: prometheus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus
        ports:
        - containerPort: 9090
        volumeMounts:
        - name: prometheus-config
          mountPath: /etc/prometheus/prometheus.yml
          subPath: prometheus.yml
      volumes:
      - name: prometheus-config
        configMap:
          name: prometheus-config
```

Crea un archivo `prometheus-configmap.yaml` para la configuración:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  labels:
    app: prometheus
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    scrape_configs:
      - job_name: 'kubernetes-services'
        kubernetes_sd_configs:
          - role: service
        relabel_configs:
          - source_labels: [__meta_kubernetes_service_label_app]
            action: keep
            regex: .+ # Mantén servicios con la etiqueta "app" definida
```

Aplica los recursos:
```bash
kubectl apply -f prometheus-configmap.yaml
kubectl apply -f prometheus-deployment.yaml
```

### 3.2 Exponer Prometheus
Crea un archivo `prometheus-service.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  labels:
    app: prometheus
spec:
  type: NodePort
  ports:
    - port: 9090
      targetPort: 9090
      protocol: TCP
      nodePort: 30090
  selector:
    app: prometheus
```

Aplica el servicio:
```bash
kubectl apply -f prometheus-service.yaml
```
Accede a Prometheus utilizando `<node-ip>:30090` en tu navegador.

## Paso 4: Desplegar una API en el Cluster

### 4.1 Crear y Subir la Imagen de la API al Registry
En tu máquina local o en uno de los nodos con acceso al cluster, sube la imagen al registry del cluster. Sustituye `<master-ip>` y `<node-port>` con la dirección IP del nodo maestro y el puerto asignado al servicio del registry:
```bash
docker build -t <master-ip>:32000/<service-name>:latest .
docker push <master-ip>:32000/<service-name>:latest
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
Accede a la API en `<node-ip>:<node-port>`.

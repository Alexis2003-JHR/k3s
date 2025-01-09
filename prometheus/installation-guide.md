# Requisitos preliminares

## 1. Configuración de `config.yaml` para Añadir Flags Adicionales

En algunos casos, es necesario configurar K3s con flags adicionales para habilitar métricas o realizar ajustes específicos. Esta configuración se realiza editando el archivo `config.yaml` para K3s. Si este archivo no existe, debe ser creado manualmente.

### Pasos:

1. **Localiza o crea el archivo `config.yaml`**:
   - Por defecto, este archivo se encuentra en:
     ```bash
     /etc/rancher/k3s/config.yaml
     ```
   - Si no existe, créalo manualmente:
     ```bash
     sudo nano /etc/rancher/k3s/config.yaml
     ```

2. **Añade las siguientes configuraciones**:
   ```yaml
   write-kubeconfig-mode: "644"
   extra-server-args:
     - "--no-deploy=servicelb"
     - "--no-deploy=traefik"
     - "--kube-controller-manager-arg=bind-address=0.0.0.0"
     - "--kube-proxy-arg=metrics-bind-address=0.0.0.0"
     - "--kube-scheduler-arg=bind-address=0.0.0.0"
   ```

3. **Reinicia el servicio de K3s**:
   - Una vez editado o creado el archivo, reinicia K3s para que los cambios surtan efecto:
     ```bash
     sudo systemctl restart k3s
     ```

4. **Verifica que K3s esté funcionando correctamente**:
   ```bash
   sudo k3s kubectl get nodes
   ```

---

## 2. Instalar HELM

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```
---
---
# Configuracion e instalación

Verifica que puedas comunicarte con el cluster
```bash
kubectl get nodes
```
```bash
NAME     STATUS   ROLES                       AGE   VERSION
k3s-01   Ready    control-plane,etcd,master   10h   v1.23.4+k3s1
```
Verifica que heml esté instalado
```bash
helm version
```
```bash
version.BuildInfo{Version:"v3.8.0", GitCommit:"d14138609b01886f544b2025f5000351c9eb092e", GitTreeState:"clean", GoVersion:"go1.17.5"}
```
Agrega agrega Prometheus al repositorio de helm:
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```
Actualiza el repositorio
```bash
helm repo update
```
Crea un namespace para servicios de monitoreo del cluster
```bash
kubectl create namespace monitoring
```
Crea archivos que contendran las credenciales de acceso de Grafana para añadirlas a Kubernetes Secret
```bash
echo -n 'adminuser' > ./admin-user # change your username
echo -n 'p@ssword!' > ./admin-password # change your password
```
Crea el Secret en Kubernetes, estos valores serán utilizados en el archivo de configuracion values.yaml
```bash
kubectl create secret generic grafana-admin-credentials --from-file=./admin-user --from-file=admin-password -n monitoring
```
Y podrás ver:
```bash
secret/grafana-admin-credentials created
```
Verifica el secret creado
```bash
kubectl describe secret -n monitoring grafana-admin-credentials
```
Y verás algo como
```bash
Name:         grafana-admin-credentials
Namespace:    monitoring
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
admin-password:  9 bytes
admin-user:      9 bytes
```
Para ver el username guardado en el secret
```bash
kubectl get secret -n monitoring grafana-admin-credentials -o jsonpath="{.data.admin-user}" | base64 --decode
```
Y verás algo como:
```bash
adminuser%
```
Para ver el password guardado en el secret
```bash
kubectl get secret -n monitoring grafana-admin-credentials -o jsonpath="{.data.admin-password}" | base64 --decode
```
Y verás algo como:
```bash
p@ssword!%
```
Elimina los archivos creados anteriormente
```bash
rm admin-user && rm admin-password
```
Crea el stack de servicios de Prometheus, asegurate de remplazar las IPs encontradas en el archivo values.yaml por las del cluster
```bash
helm install -n monitoring prometheus prometheus-community/kube-prometheus-stack -f values.yaml
```
Es posible que encuentres un error en la instalación similar a este:
```bash
Error: INSTALLATION FAILED: Kubernetes cluster unreachable: Get "http://localhost:8080/version": dial tcp 127.0.0.1:8080: connect: connection refused
```

## Solución al Error: "Kubernetes Cluster Unreachable"

Durante la instalación de Prometheus con Helm, puede aparecer el siguiente error:
```bash
Kubernetes cluster unreachable: Get "http://localhost:8080/version": dial tcp 127.0.0.1:8080: connect: connection refused
```

Esto ocurre porque la configuración del cluster no está correctamente referenciada en el archivo de kubeconfig.

### Solución:

1. **Localiza el archivo `k3s.yaml` generado por K3s**:
   - Por defecto, este archivo se encuentra en:
     ```bash
     /etc/rancher/k3s/k3s.yaml
     ```

2. **Copia este archivo al directorio de configuración del usuario**:
   ```bash
   sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
   ```

3. **Verifica la conectividad al cluster**:
   ```bash
   kubectl get nodes
   ```

Con esto, Helm podrá acceder al cluster sin problemas.

## Configuración de Grafana como NodePort

Por defecto, Grafana puede estar configurado como un servicio `ClusterIP`, lo cual restringe el acceso al servicio únicamente dentro del cluster. Para habilitar el acceso desde fuera del cluster, se debe cambiar a `NodePort`.

### Pasos:

1. **Exporta la configuración del servicio de Grafana a un archivo YAML**:
   ```bash
   kubectl get service -n monitoring grafana -o yaml > grafana-service.yaml
   ```

2. **Edita el archivo YAML**:
   - Abre el archivo con `nano`:
     ```bash
     nano grafana-service.yaml
     ```
   - Busca la siguiente línea:
     ```yaml
     type: ClusterIP
     ```
   - Cámbiala por:
     ```yaml
     type: NodePort
     ```

3. **Aplica la nueva configuración**:
   ```bash
   kubectl apply -f grafana-service.yaml
   ```

4. **Verifica que Grafana esté configurado como NodePort**:
   ```bash
   kubectl get service -n monitoring grafana
   ```

   Busca el puerto asignado bajo la columna `PORT(S)`. Ahora podrás acceder a Grafana en la dirección IP de cualquier nodo del cluster y el puerto asignado.


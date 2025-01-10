# Instalación y configuracion de Promtail + Loki

## 1. Instalación de Promtail con values especificos
```bash
helm repo add grafana https://grafana.github.io/helm-charts
```
```bash
helm repo update
```
```bash
helm upgrade --install promtail grafana/promtail -f /loki/promtail/values.yaml -n monitoring
```
Revisa que los despliegues se hayan efectuado correctamente:
```bash
kubectl get daemonset/promtail -n monitoring
```
```bash
NAME       DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
promtail   1         1         1       1            1           <none>          36s
```
## 2. Instalación de Loki:
```bash
helm upgrade --install loki grafana/loki-distributed -n monitoring
```
Revisa que estén correctamente desplegados los componentes y estén listos para su uso:
```bash
kubectl get all -n monitoring
```
Finalmente podrás visualizar en la interfaz de Grafana en el apartado **Connections/Data sources** que se encuentra el componente Loki y realizar tus consultas en el apartado **Explore**.
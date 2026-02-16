````md
# Kubernetes — Comandos de verificación, salud del clúster y pruebas (HA, reparto de carga, resiliencia)

Este documento recopila **comandos prácticos** para:
- consultar el **estado del clúster**
- diagnosticar problemas (red, DNS, scheduling, storage)
- comprobar **alta disponibilidad operativa** (resiliencia ante fallo de nodo/pod)
- validar **reparto de carga** (Service/LoadBalancer, balanceo entre réplicas)
- validar **autoscaling** (HPA + metrics)

> Nota: En tu lab tienes 1 control-plane (no HA de etcd/control-plane). Las pruebas de “HA” aquí se enfocan en **alta disponibilidad de la aplicación** (réplicas, self-healing, balanceo), no en HA del plano de control.

---

## 1) Estado general del clúster (salud rápida)

### Nodos y versiones
```bash
kubectl get nodes -o wide
kubectl describe node <NODO>
kubectl get nodes -L monitor
````

### Pods globales (ver fallos rápido)

```bash
kubectl get pods -A -o wide
kubectl get pods -A --field-selector=status.phase!=Running
```

### Eventos (lo primero cuando algo “no arranca”)

```bash
kubectl get events -A --sort-by=.lastTimestamp | tail -n 50
```

### Componentes críticos (kube-system)

```bash
kubectl -n kube-system get pods -o wide
kubectl -n kube-system get ds,deploy -o wide
```

---

## 2) Diagnóstico de red / DNS

### Comprobar CoreDNS

```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=50
```

### Probar DNS desde un pod (herramienta rápida)

```bash
kubectl run -it --rm dns-test --image=busybox:1.36 --restart=Never -- nslookup kubernetes.default.svc.cluster.local
```

### Probar conectividad HTTP entre pods (debug)

```bash
kubectl run -it --rm curl --image=curlimages/curl:8.5.0 --restart=Never -- sh
# dentro del pod:
curl -sS http://web-php-svc.web.svc.cluster.local
```

### Ver endpoints reales detrás de un Service (muy útil para balanceo)

```bash
kubectl -n web get svc web-php-svc -o wide
kubectl -n web get endpoints web-php-svc -o wide
kubectl -n web describe svc web-php-svc
```

---

## 3) Diagnóstico de scheduling y afinidad (nodeSelector, labels)

### Ver en qué nodo corre cada pod

```bash
kubectl -n web get pods -o wide
kubectl -n monitoring get pods -o wide | egrep -i "prometheus|grafana|alertmanager"
```

### Comprobar por qué un pod no se programa

```bash
kubectl -n <ns> describe pod <pod>
# mirar "Events:" (taints, recursos insuficientes, nodeSelector no coincide, etc.)
```

### Ver labels de un nodo (para nodeSelector)

```bash
kubectl get node monitor --show-labels | tr ',' '\n' | head
kubectl get nodes -L monitor
```

### Probar nodeSelector (pod de prueba)

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-monitor
spec:
  nodeSelector:
    monitor: "true"
  containers:
  - name: pause
    image: registry.k8s.io/pause:3.9
EOF

kubectl get pod test-monitor -o wide
kubectl delete pod test-monitor
```

---

## 4) Estado de la aplicación web (deployments / rollout / replicas)

### Estado general del namespace web

```bash
kubectl -n web get deploy,rs,pods,svc,hpa -o wide
```

### Rollout y troubleshooting del deployment

```bash
kubectl -n web rollout status deploy/web-php
kubectl -n web rollout history deploy/web-php
kubectl -n web describe deploy web-php
```

### Ver logs de pods (seleccionando por label)

```bash
kubectl -n web logs -l app=web-php --tail=50
kubectl -n web logs -l app=web-php -f
```

### Entrar a un pod (debug)

```bash
POD=$(kubectl -n web get pod -l app=web-php -o jsonpath='{.items[0].metadata.name}')
kubectl -n web exec -it $POD -- bash
```

---

## 5) Pruebas de Alta Disponibilidad (resiliencia de la app)

> Objetivo: demostrar que, aunque falle un pod o un nodo worker, la app sigue respondiendo gracias a réplicas + Service + self-healing.

### 5.1 “Self-healing” al borrar un pod

1. Ver pods actuales:

```bash
kubectl -n web get pods -o wide
```

2. Borrar 1 pod:

```bash
kubectl -n web delete pod <pod_name>
```

3. Observar cómo se crea otro:

```bash
kubectl -n web get pods -w
```

✅ Esperado: el Deployment crea un pod nuevo automáticamente.

---

### 5.2 Resiliencia ante fallo de un worker (simulado)

**Opción A: drenar nodo (mantenimiento)**

```bash
kubectl drain worker1 --ignore-daemonsets --delete-emptydir-data
kubectl -n web get pods -o wide
```

✅ Esperado: los pods web se reubican en otros nodos disponibles.

Restaurar:

```bash
kubectl uncordon worker1
```

**Opción B: apagar VM worker1**

* Apaga `worker1` en VirtualBox
* Ver estado:

```bash
kubectl get nodes
kubectl -n web get pods -o wide
```

✅ Esperado: la app sigue respondiendo si queda al menos 1 réplica en otro nodo.

---

## 6) Reparto de carga (Service / LoadBalancer / MetalLB)

### 6.1 Ver service y EXTERNAL-IP (MetalLB)

```bash
kubectl -n web get svc web-php-svc -o wide
kubectl -n metallb-system get pods -o wide
```

### 6.2 Probar balanceo: ver qué pod responde (cabecera / HTML)

Si tu página imprime “Pod: <nombre>” (como en tu práctica), repite peticiones y observa alternancia.

```bash
for i in {1..20}; do curl -s http://192.168.1.20 | egrep -o "Pod: [^<]+"; done | sort | uniq -c
```

✅ Esperado: aparecen varias respuestas repartidas entre pods.

> Nota: El balanceo depende del modo de kube-proxy / iptables y del cliente. A veces verás “sesgo” si hay keep-alive, DNS cache o pocas peticiones. Por eso lanzamos varias.

### 6.3 Validar endpoints detrás del service

```bash
kubectl -n web get endpoints web-php-svc -o wide
```

✅ Debe listar IPs de todos los pods web listos.

---

## 7) Pruebas de HPA (autoscaling) y métricas

### 7.1 Ver estado de metrics-server

```bash
kubectl -n kube-system get pods -l k8s-app=metrics-server -o wide
kubectl top nodes
kubectl top pods -n web
```

### 7.2 Ver estado de HPA

```bash
kubectl -n web get hpa -o wide
kubectl -n web describe hpa web-php-hpa
```

### 7.3 Generar carga para forzar scale up (2 métodos)

**Método A: pod “loader” dentro del cluster**

```bash
kubectl -n web run loader --image=busybox:1.36 --restart=Never -it -- sh
# dentro:
while true; do wget -q -O- http://web-php-svc; done
```

**Método B: desde un nodo externo (ej. worker2)**

```bash
while true; do curl -s http://192.168.1.20 >/dev/null; done
```

### 7.4 Observar autoscaling en vivo

```bash
watch -n 5 'kubectl -n web get hpa; echo; kubectl -n web get pods -o wide'
```

✅ Esperado: suben réplicas cuando CPU pasa el objetivo y bajan tras un tiempo sin carga.

---

## 8) Storage (PV/PVC) y NFS — verificación

### PV/PVC estado

```bash
kubectl get pv
kubectl -n web get pvc
kubectl -n web describe pvc pvc-nfs-web
```

### Ver montajes en el pod

```bash
kubectl -n web describe pod -l app=web-php | egrep -i "Mounts|Volumes|nfs|pvc|claim"
```

### Ver export NFS desde un nodo

```bash
showmount -e 192.168.1.9
```

---

## 9) Monitoring (Prometheus/Grafana) — estado y validaciones

### Pods principales

```bash
kubectl -n monitoring get pods -o wide | egrep -i "prometheus|grafana|alertmanager|operator|kube-state|node-exporter"
```

### Verificar que Prometheus/Grafana caen en nodo monitor

```bash
kubectl -n monitoring get pods -o wide | egrep -i "prometheus|grafana"
# comprobar columna NODE = monitor
```

### Services

```bash
kubectl -n monitoring get svc -o wide | egrep -i "grafana|prometheus|alertmanager"
```

### Obtener password admin de Grafana

```bash
kubectl get secret -n monitoring -l app.kubernetes.io/name=grafana \
  -o jsonpath="{.items[0].data.admin-password}" | base64 -d; echo
```

---

## 10) Comandos “de oro” para troubleshooting rápido

### 10.1 Describe + Events (si algo falla)

```bash
kubectl -n <ns> describe pod <pod>
kubectl -n <ns> get events --sort-by=.lastTimestamp | tail -n 30
```

### 10.2 Logs de un pod

```bash
kubectl -n <ns> logs <pod> --tail=100
kubectl -n <ns> logs <pod> -f
```

### 10.3 Ver recursos y consumo (si hay métricas)

```bash
kubectl top nodes
kubectl top pods -A | head
```

### 10.4 Ver “por qué no schedulea”

```bash
kubectl -n <ns> describe pod <pod> | egrep -i "nodeSelector|taint|tolerations|Insufficient|FailedScheduling|Events" -n
```

---

## 11) Checklist de “prueba final” (lo que se suele pedir en prácticas)

1. **Nodos Ready**:

```bash
kubectl get nodes
```

2. **DNS OK**:

```bash
kubectl run -it --rm dns-test --image=busybox:1.36 --restart=Never -- nslookup kubernetes.default
```

3. **Web responde por LoadBalancer**:

```bash
kubectl -n web get svc web-php-svc -o wide
curl -s http://192.168.1.20 | head
```

4. **Balanceo funciona (varias réplicas)**:

```bash
for i in {1..20}; do curl -s http://192.168.1.20 | egrep -o "Pod: [^<]+"; done | sort | uniq -c
```

5. **HPA operativo**:

```bash
kubectl -n web get hpa
kubectl top pods -n web
```

6. **Resiliencia** (borrar pod y se recrea):

```bash
kubectl -n web delete pod -l app=web-php --force --grace-period=0
kubectl -n web get pods -w
```

7. **Monitorización OK (Grafana accesible)**:

```bash
kubectl -n monitoring get pods -o wide | egrep -i "grafana|prometheus"
```

---

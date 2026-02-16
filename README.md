# Actividad 3 — Kubernetes (kubeadm) + Web + NFS + HPA + MetalLB + Ansible + Prometheus/Grafana

> Proyecto realizado siguiendo los requisitos de **actividad_3**: clúster con **kubeadm (1 master + 2 workers)**, despliegue de **servidor web (Apache/PHP)** con **YAML**, **HPA** (requiere **metrics-server**), **MetalLB** para `LoadBalancer`, y **NFS** para persistencia compartida por todos los pods web. :contentReference[oaicite:0]{index=0}

---

## 1) Topología y red

### VMs / IPs (red NAT “red-k8s”)
- **master**: `192.168.1.6`
- **worker1**: `192.168.1.7`
- **worker2**: `192.168.1.8`
- **nfs**: `192.168.1.9`
- **ansible**: `192.168.1.5` (nodo de automatización)
- **monitor**: `192.168.1.4` (nuevo worker para Prometheus/Grafana)

> Nota: La actividad define explícitamente el uso de red NAT y las IPs/recursos orientativos para las VMs. :contentReference[oaicite:1]{index=1}

---

## 2) Kubernetes con kubeadm (cluster base)

### 2.1 Prerrequisitos (en master y workers)
Comprobaciones habituales:
- Conectividad (gateway e Internet): `ip r`, `ping -c 1 8.8.8.8`
- Swap deshabilitado: `free -h | grep -i swap` (Swap: 0)
- Forwarding IPv4 habilitado: `sysctl net.ipv4.ip_forward` (debe ser 1)

Servicios:
- `containerd` instalado y activo
- `kubelet` habilitado y arrancado (queda en “activating” hasta que el nodo se une/arranca el control-plane)

#### containerd y SystemdCgroup
Se verificó / ajustó:
- `/etc/containerd/config.toml` → `SystemdCgroup = true`

---

### 2.2 Inicialización del control-plane
En **master**:
- `kubeadm init ...` (con CIDR compatible con Flannel)
- Configurar kubeconfig para kubectl (usuario administrador)

> En kubeadm, el init genera también el comando `kubeadm join` para los workers. :contentReference[oaicite:2]{index=2}

---

### 2.3 CNI (Flannel)
Se instaló **Flannel** y se verificó:
- Namespace `kube-flannel`
- DaemonSet de flannel ejecutándose en los nodos

---

### 2.4 Join de workers
En **worker1** y **worker2**:
- `kubeadm join 192.168.1.6:6443 --token ... --discovery-token-ca-cert-hash ...`

Verificación en **master**:
- `kubectl get nodes -o wide` → todos `Ready`

---

## 3) Despliegue web con NFS (PV/PVC) — Requisito actividad_3

### 3.1 NFS server (nodo `nfs` 192.168.1.9)
Export:
- Directorio: `/srv/nfs/web`
- Red permitida: `192.168.1.0/24`

Comandos típicos:
- `exportfs -v`
- `showmount -e 192.168.1.9`
- Servicios: `rpcbind`, `nfs-server`

> Requisito: “Añadir nodo NFS… y todos los pods deben servir la misma página almacenada en NFS”. :contentReference[oaicite:3]{index=3}

---

### 3.2 Namespace “web”
`manifests/web/01-namespace.yaml`
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: web
````

> Los recursos se organizan por **namespace**; es separación lógica y afecta a nombres/operaciones. 

---

### 3.3 PersistentVolume (PV) NFS

`manifests/web/02-pv-nfs-web.yaml`

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs-web
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: 192.168.1.9
    path: /srv/nfs/web
```

### 3.4 PersistentVolumeClaim (PVC) NFS

`manifests/web/03-pvc-nfs-web.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-nfs-web
  namespace: web
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  volumeName: pv-nfs-web
```

Verificación:

* `kubectl get pv`
* `kubectl -n web get pvc`
* `kubectl -n web describe pvc pvc-nfs-web`

---

### 3.5 Deployment web (Apache/PHP) usando PVC NFS

`manifests/web/04-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-php
  namespace: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-php
  template:
    metadata:
      labels:
        app: web-php
    spec:
      containers:
      - name: php-apache
        image: php:8.2-apache
        ports:
        - containerPort: 80
        volumeMounts:
        - name: webcontent
          mountPath: /var/www/html
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 300m
            memory: 256Mi
      volumes:
      - name: webcontent
        persistentVolumeClaim:
          claimName: pvc-nfs-web
```

> Recordatorio conceptual: en Kubernetes los manifests se basan en `metadata/spec/status`: **spec** es lo deseado, **status** lo observado (lo que el clúster ha conseguido). 

---

### 3.6 Service para exponer la web

Inicialmente (NodePort), y después con MetalLB (LoadBalancer).

`manifests/web/05-service-lb.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-php-svc
  namespace: web
spec:
  type: LoadBalancer
  selector:
    app: web-php
  ports:
  - port: 80
    targetPort: 80
```

Verificación:

* `kubectl -n web get svc -o wide`
* `curl -s http://<EXTERNAL-IP> | head`

---

## 4) Metrics-server (necesario para HPA)

### 4.1 Instalación

* Se desplegó metrics-server en `kube-system`

### 4.2 Problema encontrado

Errores de scrape a kubelet por TLS:

* `x509: cannot validate certificate ... doesn't contain any IP SANs`

### 4.3 Fix aplicado

Se ajustó el deployment de metrics-server (args), incluyendo:

* `--kubelet-insecure-tls`
* `--kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname`

Verificación:

* `kubectl top nodes`
* `kubectl top pods -n web`

> Metrics-server y `kubectl top` aparecen como parte habitual de observabilidad y troubleshooting. 

---

## 5) HPA (Horizontal Pod Autoscaler)

### 5.1 Creación del HPA

`manifests/web/06-hpa.yaml`

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-php-hpa
  namespace: web
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-php
  minReplicas: 2
  maxReplicas: 6
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

### 5.2 Problema encontrado

El HPA marcaba “invalid metrics / missing request for cpu”:

* **Causa**: faltaban `resources.requests.cpu` (y/o memory) en el container del deployment.
* **Solución**: añadir `requests/limits` al Deployment (ver sección 3.5) y esperar rollout.

Verificación:

* `kubectl -n web get hpa`
* `kubectl -n web describe hpa web-php-hpa`

> El autoscaling depende de métricas (vía metrics-server) y los “resource requests” son fundamentales para decisiones del scheduler/autoscaling.

---

## 6) MetalLB para `LoadBalancer` (requisito actividad_3)

### 6.1 Instalación

* Namespace `metallb-system`
* Pods controller/speaker en Running

### 6.2 Pool y L2Advertisement

`manifests/metallb/metallb-pool.yaml`

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: pool-k8s
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.20-192.168.1.30
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-k8s
  namespace: metallb-system
spec:
  ipAddressPools:
  - pool-k8s
```

### 6.3 Web service como LoadBalancer

* `kubectl -n web patch svc web-php-svc -p '{"spec":{"type":"LoadBalancer"}}'`
* Obtuvo `EXTERNAL-IP` (ej. `192.168.1.20`)
* Prueba: `curl http://192.168.1.20`

> MetalLB es requisito para poder usar `Service type LoadBalancer` en entornos on-prem/lab. 

---

## 7) Etiquetas y nodeSelector (base para “monitor node”)

Se utilizó el mecanismo de **labels** + **nodeSelector** para forzar pods a nodos concretos (luego usado para Prometheus/Grafana). La idea:

1. Etiquetar nodo:

   * `kubectl label node monitor monitor=true`
2. En el Pod/Deployment:

   * `nodeSelector: { monitor: "true" }`

> `nodeSelector` restringe el scheduling a nodos que tengan esas labels (hard requirement).

---

# 8) Automatización con Ansible (sobre el clúster ya creado)

> La actividad también pide una parte con Ansible (sin Kubespray) y define VM “ansible” separada. 

## 8.1 Estructura de proyecto (en nodo ansible)

Ruta base: `~/k8s-ops/`

Ejemplo de estructura:

```
k8s-ops/
├─ ansible.cfg
├─ inventories/
│  └─ hosts.ini
├─ playbooks/
│  ├─ 01-cluster-check.yml
│  ├─ 02-ensure-web.yml
│  ├─ 03-rolling-restart-web.yml
│  ├─ 04-ensure-metallb-pool.yml
│  └─ 05-export-manifests.yml
├─ manifests/
│  ├─ web/               # namespace/pv/pvc/deploy/svc/hpa
│  └─ metallb/           # pool + L2
└─ exports/              # recursos exportados desde el cluster
```

## 8.2 Inventario

`inventories/hosts.ini` (ejemplo)

```ini
[masters]
master ansible_host=192.168.1.6

[workers]
worker1 ansible_host=192.168.1.7
worker2 ansible_host=192.168.1.8
monitor ansible_host=192.168.1.4

[nfs]
nfs ansible_host=192.168.1.9

[all:vars]
ansible_user=root
ansible_python_interpreter=/usr/bin/python3
```

> Nota: se corrigieron warnings iniciales por “group y host con el mismo nombre” cambiando a grupos en plural.

## 8.3 ansible.cfg (stdout_callback)

Se detectó error: `Invalid callback for stdout specified: yaml`
Se arregló cambiando el callback a `default`.

Comando usado (edición “in-place”):

```bash
sed -i 's/^stdout_callback *=.*/stdout_callback = default/' ansible.cfg
```

---

## 8.4 Playbook 01 — Check básico del cluster

`playbooks/01-cluster-check.yml` (ejemplo funcional)

```yaml
---
- name: Check básico del cluster Kubernetes
  hosts: localhost
  gather_facts: false

  tasks:
    - name: Nodos
      command: kubectl get nodes -o wide
      register: nodes

    - name: Mostrar nodos
      debug:
        var: nodes.stdout_lines

    - name: Estado app web
      command: kubectl -n web get deploy,svc,hpa,pods -o wide
      register: web

    - name: Mostrar app web
      debug:
        var: web.stdout_lines

    - name: Top nodes
      command: kubectl top nodes
      register: top_nodes
      failed_when: false

    - name: Mostrar top nodes
      debug:
        var: top_nodes.stdout_lines
```

---

## 8.5 Playbook 02 — Asegurar recursos de la app web (idempotente)

`playbooks/02-ensure-web.yml`

```yaml
---
- name: Asegurar recursos de la app web (namespace, pv/pvc, deploy, svc, hpa)
  hosts: localhost
  gather_facts: false

  tasks:
    - name: Aplicar manifests web (idempotente)
      command: kubectl apply -f manifests/web/
      args:
        chdir: "{{ playbook_dir }}/.."
      register: apply_web

    - name: Mostrar salida apply
      debug:
        var: apply_web.stdout_lines

    - name: Ver estado
      command: kubectl -n web get deploy,svc,hpa,pods -o wide
      register: state

    - name: Mostrar estado
      debug:
        var: state.stdout_lines
```

> Concepto: aplicar manifests desde Ansible es un patrón común (vía `k8s` module o ejecutando `kubectl apply` desde el controlador).

---

## 8.6 Playbook 03 — Rolling restart del deployment

`playbooks/03-rolling-restart-web.yml`

```yaml
---
- name: Rolling restart del deployment web-php
  hosts: localhost
  gather_facts: false

  tasks:
    - name: Reiniciar deployment (rolling restart)
      command: kubectl -n web rollout restart deploy/web-php
      register: restart_out

    - name: Mostrar salida restart
      debug:
        var: restart_out.stdout_lines

    - name: Esperar a que termine el rollout
      command: kubectl -n web rollout status deploy/web-php
      register: rollout

    - name: Mostrar estado final
      debug:
        var: rollout.stdout_lines
```

---

## 8.7 Playbook 04 — Asegurar MetalLB pool + L2

`playbooks/04-ensure-metallb-pool.yml`

```yaml
---
- name: Asegurar MetalLB pool + L2
  hosts: localhost
  gather_facts: false

  tasks:
    - name: Aplicar MetalLB pool (idempotente)
      command: kubectl apply -f manifests/metallb/metallb-pool.yaml
      args:
        chdir: "{{ playbook_dir }}/.."
      register: out

    - name: Mostrar apply
      debug:
        var: out.stdout_lines

    - name: Ver pods metallb-system
      command: kubectl -n metallb-system get pods -o wide
      register: pods

    - name: Mostrar pods
      debug:
        var: pods.stdout_lines
```

---

## 8.8 Playbook 05 — Exportar manifests “reales” desde el cluster

Objetivo: guardar recursos tal como están en el clúster (útil para auditoría/backup).

Ejemplo:
`playbooks/05-export-manifests.yml`

```yaml
---
- name: Exportar recursos Kubernetes a YAML
  hosts: localhost
  gather_facts: false

  vars:
    outdir: "{{ playbook_dir }}/../exports"

  tasks:
    - name: Crear carpeta exports
      file:
        path: "{{ outdir }}"
        state: directory

    - name: Export ns web
      command: kubectl get ns web -o yaml
      register: ns_web
    - copy:
        dest: "{{ outdir }}/ns-web.yaml"
        content: "{{ ns_web.stdout }}"

    - name: Export PV/PVC web
      command: kubectl get pv pv-nfs-web -o yaml
      register: pv
    - copy:
        dest: "{{ outdir }}/pv-nfs-web.yaml"
        content: "{{ pv.stdout }}"

    - command: kubectl -n web get pvc pvc-nfs-web -o yaml
      register: pvc
    - copy:
        dest: "{{ outdir }}/pvc-nfs-web.yaml"
        content: "{{ pvc.stdout }}"

    - name: Export deploy/svc/hpa web
      command: kubectl -n web get deploy web-php -o yaml
      register: dep
    - copy:
        dest: "{{ outdir }}/web-deployment.yaml"
        content: "{{ dep.stdout }}"

    - command: kubectl -n web get svc web-php-svc -o yaml
      register: svc
    - copy:
        dest: "{{ outdir }}/web-service.yaml"
        content: "{{ svc.stdout }}"

    - command: kubectl -n web get hpa web-php-hpa -o yaml
      register: hpa
    - copy:
        dest: "{{ outdir }}/web-hpa.yaml"
        content: "{{ hpa.stdout }}"
```

En la práctica se generaron ficheros exportados, por ejemplo:

* `metallb-pool-k8s.yaml`
* `metallb-l2-k8s.yaml`
* `ns-web.yaml`
* `pv-nfs-web.yaml`
* `pvc-nfs-web.yaml`
* `web-deployment.yaml`
* `web-service.yaml`
* `web-hpa.yaml`

---

# 9) Prometheus + Grafana (kube-prometheus-stack) en nodo monitor

## 9.1 Añadir nodo monitor como nuevo worker

* Se creó VM `monitor` (IP `192.168.1.4`)
* Se instaló `containerd`, `kubelet`, `kubeadm` y se ejecutó `kubeadm join ...`
* Verificación:

  * `kubectl get nodes -o wide` → `monitor Ready`

## 9.2 Label y nodeSelector (monitor dedicado)

Label:

```bash
kubectl label node monitor monitor=true
kubectl get nodes -L monitor
```

> Los pods con `nodeSelector` solo se programan en nodos que cumplan esa label.

## 9.3 Instalación de Helm (en master)

Dependencias necesarias:

* `git`, `tar` (para que el script pueda descargar y extraer el binario)

Instalación:

```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

## 9.4 Despliegue kube-prometheus-stack forzando Prometheus/Grafana a `monitor`

Namespace:

```bash
kubectl create ns monitoring
```

Values:
`k8s-manifests/monitoring-values.yaml`

```yaml
prometheus:
  prometheusSpec:
    nodeSelector:
      monitor: "true"

grafana:
  nodeSelector:
    monitor: "true"
```

Instalación:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f k8s-manifests/monitoring-values.yaml
```

Verificación:

```bash
kubectl -n monitoring get pods -o wide | egrep -i "prometheus|grafana|alertmanager"
```

Resultado esperado:

* **prometheus** y **grafana** en `NODE=monitor`
* `node-exporter` (DaemonSet) en todos los nodos (master/workers/monitor), comportamiento típico de DaemonSet. 

## 9.5 Exponer Grafana y acceso

Se expuso Grafana (ej. mediante `Service type LoadBalancer` con MetalLB) y se validó acceso vía navegador (pantalla “Welcome to Grafana”).

Password admin (ejemplo típico):

```bash
kubectl get secret -n monitoring -l app.kubernetes.io/name=grafana \
  -o jsonpath="{.items[0].data.admin-password}" | base64 -d; echo
```

## 9.6 Dashboards importados (ejemplos)

* “Kubernetes Views Global” (ejemplo: ID 15757)
* “Node Exporter Full” (ejemplo: ID 1860)

---

# 10) Comandos de verificación (resumen)

## Estado cluster

```bash
kubectl get nodes -o wide
kubectl get pods -A
```

## App web

```bash
kubectl -n web get deploy,svc,hpa,pods -o wide
kubectl -n web describe hpa web-php-hpa
```

## Storage

```bash
kubectl get pv
kubectl -n web get pvc
```

## Métricas

```bash
kubectl top nodes
kubectl top pods -n web
```

## MetalLB

```bash
kubectl -n metallb-system get pods -o wide
kubectl -n web get svc web-php-svc -o wide
```

## Monitoring (Prom/Grafana)

```bash
kubectl -n monitoring get pods -o wide | egrep -i "prometheus|grafana|alertmanager"
kubectl -n monitoring get svc | egrep -i "grafana|prometheus"
```

---

# 11) Notas técnicas (lo más importante aprendido)

* **Manifests YAML**: Kubernetes trabaja de forma declarativa: defines el estado deseado en `spec`, y los controladores llevan el clúster hacia ese estado; `status` refleja el estado real observado. 
* **Labels/Selectors**: son clave para:

  * Services (seleccionan pods)
  * Deployments (encuentran sus pods)
  * Scheduling por nodo (nodeSelector)
* **HPA**: para CPU (%) necesitas:

  * metrics-server funcionando
  * `resources.requests.cpu` en el contenedor (si no, “missing request for cpu”).
* **DaemonSet**: recursos como `node-exporter` se despliegan automáticamente en todos (o un subconjunto) de nodos. 
* **Ansible + Kubernetes**: patrón común es “aplicar manifests” desde playbooks (módulo `k8s` o `kubectl apply`), manteniendo idempotencia y comprobaciones de estado.

```

Si quieres, también puedo **adaptar este README a tu estructura exacta** (nombres reales de carpetas/archivos) si me pegas un `tree -L 3` del repo `k8s-ops/` y de tu carpeta `k8s-manifests/` en el master.
::contentReference[oaicite:19]{index=19}
```

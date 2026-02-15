
# Kubernetes Lab (kubeadm): NFS (RWX) + Web PHP + Metrics Server + HPA + MetalLB (LoadBalancer)

Este README recopila **toda la información relevante** de la práctica: arquitectura, comandos ejecutados, manifiestos YAML creados, parches aplicados y verificaciones.

---

## 0) Arquitectura y red

### Red VirtualBox
- NAT Network: `red-k8s`
- Rango: `192.168.1.0/24`
- Gateway típico: `192.168.1.1`

### Nodos
- **master (control-plane)**: `192.168.1.6`
- **worker1**: `192.168.1.7`
- **worker2**: `192.168.1.8`
- **nfs**: `192.168.1.9`

### MetalLB
- Pool IPs: `192.168.1.20-192.168.1.30`
- IP asignada al Service web: `192.168.1.20`

---

## 1) Preparación base (master + workers)

### Hostnames
```bash
sudo hostnamectl set-hostname master    # master
sudo hostnamectl set-hostname worker1   # worker1
sudo hostnamectl set-hostname worker2   # worker2
````

### /etc/hosts (en master y workers)

```txt
192.168.1.6 master
192.168.1.7 worker1
192.168.1.8 worker2
192.168.1.9 nfs
```

### Swap OFF (requisito kubelet/kubeadm)

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
free -h | grep -i swap
```

### Kernel/network params (CNI / iptables sobre bridge)

```bash
cat <<'EOF' | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF
sudo modprobe br_netfilter

cat <<'EOF' | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sudo sysctl --system

sysctl net.ipv4.ip_forward
lsmod | grep br_netfilter
```

---

## 2) Runtime: containerd (AlmaLinux 9 minimal)

> En AlmaLinux minimal `containerd` no estaba disponible en repos por defecto, se instaló `containerd.io` desde el repo de Docker.

```bash
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf -y install containerd.io
```

### Configurar systemd cgroups

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl enable --now containerd
systemctl is-active containerd
grep -n "SystemdCgroup" /etc/containerd/config.toml
```

---

## 3) Kubernetes paquetes + ajustes OS (laboratorio)

### SELinux permissive + firewall OFF (lab)

```bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=.*/SELINUX=permissive/' /etc/selinux/config
sudo systemctl disable --now firewalld

getenforce
systemctl is-active firewalld
```

### Repo Kubernetes (v1.29) y paquetes

```bash
cat <<'EOF' | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/repodata/repomd.xml.key
EOF

sudo dnf makecache
```

* Master:

```bash
sudo dnf -y install kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

* Workers:

```bash
sudo dnf -y install kubelet kubeadm
sudo systemctl enable --now kubelet
```

> Nota: antes de `kubeadm init`, `kubelet` puede aparecer en `activating (auto-restart)`.

---

## 4) Crear cluster con kubeadm

### kubeadm init (master)

```bash
sudo kubeadm init \
  --apiserver-advertise-address=192.168.1.6 \
  --pod-network-cidr=10.244.0.0/16
```

### kubeconfig para kubectl (master)

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get nodes -o wide
```

### Instalar CNI (Flannel)

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
kubectl get nodes
kubectl get pods -A
```

### Unir workers (worker1 y worker2)

```bash
sudo kubeadm join 192.168.1.6:6443 --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

### Validación cluster

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl get pods -n kube-flannel -o wide
```

---

## 5) NFS Server (VM `nfs` 192.168.1.9)

### Instalar y configurar NFS (en nfs)

```bash
sudo dnf -y install nfs-utils
sudo mkdir -p /srv/nfs/web
sudo chmod -R 0777 /srv/nfs/web

echo "/srv/nfs/web 192.168.1.0/24(rw,sync,no_subtree_check,no_root_squash)" | sudo tee /etc/exports

sudo systemctl enable --now rpcbind
sudo systemctl enable --now nfs-server
sudo exportfs -rav

# Laboratorio: apagar firewall en nfs
sudo systemctl disable --now firewalld

rpcinfo -p localhost | head
exportfs -v
```

### Contenido web (en nfs)

```bash
cat <<'EOF' | sudo tee /srv/nfs/web/index.php
<?php
echo "<h1>Hola desde Kubernetes</h1>";
echo "<p>Pod: " . gethostname() . "</p>";
echo "<p>Fecha: " . date('c') . "</p>";
?>
EOF
```

### Comprobación desde master

```bash
ping -c 2 192.168.1.9
showmount -e 192.168.1.9
```

### Cliente NFS en nodos Kubernetes (master + workers)

```bash
sudo dnf -y install nfs-utils
```

---

## 6) Manifests YAML creados

> Carpeta recomendada:

* `~/k8s-manifests/nfs/`
* `~/k8s-manifests/web/`
* `~/k8s-manifests/metallb/`

### 6.1 Namespace `web` (opcional pero recomendado)

**web-namespace.yaml**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: web
```

Aplicar:

```bash
kubectl apply -f web-namespace.yaml
```

### 6.2 PV NFS (RWX)

**pv-nfs-web.yaml**

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

### 6.3 PVC NFS (RWX)

**pvc-nfs-web.yaml**

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
  storageClassName: ""
```

Aplicar/validar:

```bash
kubectl apply -f pv-nfs-web.yaml
kubectl apply -f pvc-nfs-web.yaml
kubectl get pv
kubectl get pvc -n web
```

### 6.4 Deployment Web PHP (monta PVC en /var/www/html)

**web-deploy.yaml**

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
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 300m
            memory: 256Mi
        volumeMounts:
        - name: webcontent
          mountPath: /var/www/html
      volumes:
      - name: webcontent
        persistentVolumeClaim:
          claimName: pvc-nfs-web
```

Aplicar/validar:

```bash
kubectl apply -f web-deploy.yaml
kubectl -n web rollout status deploy/web-php
kubectl -n web get pods -o wide
```

### 6.5 Service (final) LoadBalancer “limpio” sin NodePort

**web-svc-lb.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-php-svc
  namespace: web
spec:
  type: LoadBalancer
  allocateLoadBalancerNodePorts: false
  loadBalancerIP: 192.168.1.20
  selector:
    app: web-php
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
```

Aplicar/validar:

```bash
kubectl apply -f web-svc-lb.yaml
kubectl -n web get svc web-php-svc -o wide
curl -s http://192.168.1.20 | head
```

### 6.6 HPA (autoscaling/v2) por CPU

**hpa-web.yaml**

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

Aplicar/validar:

```bash
kubectl apply -f hpa-web.yaml
kubectl -n web get hpa
kubectl top pods -n web
```

---

## 7) Metrics Server

### Instalación (upstream)

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl -n kube-system get pods -l k8s-app=metrics-server -o wide
kubectl get apiservice v1beta1.metrics.k8s.io -o wide
```

### Error encontrado (típico lab) y solución

**Error (logs):**

* `x509: cannot validate certificate ... doesn't contain any IP SANs`
* `no metrics to serve`, readiness probe falla

**Solución aplicada (patch):**

```bash
kubectl -n kube-system patch deployment metrics-server \
  --type='json' \
  -p='[
    {"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"},
    {"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-preferred-address-types=InternalIP,Hostname,ExternalIP"}
  ]'

kubectl -n kube-system rollout status deployment/metrics-server
kubectl get apiservice v1beta1.metrics.k8s.io -o wide
kubectl top nodes
kubectl top pods -n web
```

---

## 8) MetalLB

### Instalación (upstream)

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml
kubectl get pods -n metallb-system -o wide
```

### Configuración pool + L2Advertisement

**metallb-pool.yaml**

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

Aplicar/validar:

```bash
kubectl apply -f metallb-pool.yaml
kubectl get pods -n metallb-system
kubectl -n web get svc web-php-svc -o wide
curl -s http://192.168.1.20 | head
```

---

## 9) “Limpieza” del Service (quitar NodePort si venía de NodePort)

Si el Service nació como NodePort y se cambió a LoadBalancer, puede conservar `nodePort`.

Comprobación:

```bash
kubectl -n web get svc web-php-svc -o wide
kubectl -n web get svc web-php-svc -o jsonpath='{.spec.ports[0].nodePort}{"\n"}'
```

Solución (redefinir puertos sin nodePort):

```bash
kubectl -n web patch svc web-php-svc --type='merge' -p '{
  "spec":{
    "ports":[
      {"port":80,"protocol":"TCP","targetPort":80}
    ]
  }
}'
```

---

## 10) Verificaciones finales (estado esperado)

### Cluster

```bash
kubectl get nodes -o wide
kubectl get pods -A
```

### Componentes clave

* `kube-flannel` (CNI) con DaemonSet en todos los nodos
* `metrics-server` `READY 1/1` y APIService `AVAILABLE True`
* `metallb-system` controller + speaker `1/1 Running`
* Web app en namespace `web` con:

  * Deployment `2/2 Ready`
  * PVC `Bound`
  * HPA con `TARGETS` numéricos (no `<unknown>`)
  * Service LoadBalancer con EXTERNAL-IP `192.168.1.20`

### App

```bash
kubectl -n web get deploy,pods,svc,hpa -o wide
curl -s http://192.168.1.20 | head
```

---

## 11) Puntos clave aprendidos (resumen)

* `swapoff` es requisito práctico para kubelet/kubeadm.
* `ip_forward` + `br_netfilter` son críticos para red de pods y Services.
* Tras `kubeadm init`, CoreDNS puede estar Pending hasta instalar CNI.
* NFS: si `showmount` falla con RPC, revisar `rpcbind`, `nfs-server`, firewall.
* HPA por CPU requiere `resources.requests.cpu`.
* Metrics Server en labs kubeadm suele fallar por `x509` y se corrige con flags (`--kubelet-insecure-tls`).
* MetalLB permite `Service: LoadBalancer` en bare-metal/VirtualBox, asignando IPs de un pool.

```
```

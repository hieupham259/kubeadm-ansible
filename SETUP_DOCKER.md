# SETUP_DOCKER — Kubernetes v1.35.0 (1 master + 2 worker) với Docker Engine (cri-dockerd)

Tài liệu này mô tả các bước **chính xác** để dựng Kubernetes cluster **v1.35.0** gồm **1 master** và **2 worker**, dùng **Docker Engine** thông qua **cri-dockerd**.

> Vì sao cần cri-dockerd?
>
> Docker Engine **không implement CRI**, trong khi kubelet yêu cầu container runtime phải cung cấp CRI. Từ Kubernetes 1.24 dockershim bị gỡ khỏi kubelet, nên nếu dùng Docker Engine bạn phải cài thêm **cri-dockerd** và dùng socket:
>
> `unix:///var/run/cri-dockerd.sock`

> Ghi chú về repo này
>
>- Code hiện tại trong [roles/kubernetes/master/tasks/init.yml](roles/kubernetes/master/tasks/init.yml) và [roles/kubernetes/node/tasks/join.yml](roles/kubernetes/node/tasks/join.yml) đang hardcode CRI socket dạng `/var/run/{{ container_runtime }}/{{ container_runtime }}.sock`.
>- Nếu bạn set `container_runtime: docker` theo kiểu cũ thì socket đó sẽ **không đúng** cho Kubernetes 1.35.
>
> Vì vậy: với Kubernetes 1.35, **không nên chạy thẳng `site.yaml`** của repo này cho Docker option.

---

## A. Chuẩn bị

Topology:
- `master-1`: ví dụ `10.0.0.10`
- `worker-1`: ví dụ `10.0.0.11`
- `worker-2`: ví dụ `10.0.0.12`

Khuyến nghị OS: Ubuntu 22.04/24.04 hoặc tương đương.

---

## B. Cấu hình OS chung (trên cả 3 node)

1) Tắt swap:

```bash
sudo swapoff -a
sudo sed -i.bak -r '/\sswap\s/s/^/#/' /etc/fstab
```

2) sysctl/kernel modules:

```bash
cat <<'EOF' | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
overlay
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<'EOF' | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

---

## C. Cài Docker Engine + cri-dockerd (trên cả 3 node)

### C1) Cài Docker Engine
Cài theo hướng dẫn chính thức Docker cho distro của bạn.

Sau khi cài xong, verify:

```bash
sudo systemctl enable --now docker
sudo docker version
```

### C2) Cài cri-dockerd
Cài theo hướng dẫn của dự án cri-dockerd (Mirantis). Mục tiêu là service chạy và tạo socket:

- `unix:///var/run/cri-dockerd.sock`

Verify:

```bash
sudo systemctl enable --now cri-docker
sudo test -S /var/run/cri-dockerd.sock && echo OK
```

> Tên service có thể là `cri-docker` hoặc `cri-dockerd` tùy cách cài.

---

## D. Cài kubeadm/kubelet/kubectl v1.35.0 (trên cả 3 node)

Các bước dưới bám theo hướng dẫn chính thống Kubernetes v1.35 (pkgs.k8s.io).

### D1) Ubuntu/Debian

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update

# Chọn đúng patch version 1.35.0 bằng cách xem available versions:
apt-cache madison kubeadm | head

# Ví dụ:
# sudo apt-get install -y kubelet=1.35.0-1.1 kubeadm=1.35.0-1.1 kubectl=1.35.0-1.1
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

sudo systemctl enable --now kubelet
```

---

## E. Khởi tạo control-plane (chỉ trên master)

1) Kéo images trước:

```bash
sudo kubeadm config images pull --kubernetes-version v1.35.0 --cri-socket unix:///var/run/cri-dockerd.sock
```

2) `kubeadm init`:

```bash
sudo kubeadm init \
  --kubernetes-version v1.35.0 \
  --apiserver-advertise-address 10.0.0.10 \
  --service-cidr 10.96.0.0/12 \
  --pod-network-cidr 10.244.0.0/16 \
  --cri-socket unix:///var/run/cri-dockerd.sock
```

3) Setup kubeconfig:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

4) Lấy lệnh join:

```bash
kubeadm token create --print-join-command
```

---

## F. Join 2 worker (trên worker-1 và worker-2)

Dùng output từ bước E4 và thêm CRI socket:

```bash
sudo kubeadm join <MASTER_IP>:6443 --token <...> --discovery-token-ca-cert-hash sha256:<...> \
  --cri-socket unix:///var/run/cri-dockerd.sock
```

---

## G. Cài CNI (khuyến nghị Calico)

Trên master:

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```

Không khuyến nghị dùng flannel/canal templates trong repo này cho Kubernetes mới vì có API đã bị xóa.

---

## H. Verify

```bash
kubectl get nodes -o wide
kubectl get pods -n kube-system
```

---

## I. Nếu muốn dùng Ansible trong repo này cho Docker + v1.35 (không sửa file cũ)

Bạn sẽ cần **tạo thêm file mới** để tránh các phần hardcode/legacy trong repo hiện tại:

- Playbook mới: `site-k8s135-docker.yaml`
- Role mới để cài Kubernetes từ pkgs.k8s.io
- Role mới để cài cri-dockerd và expose biến `cri_socket=unix:///var/run/cri-dockerd.sock`
- Role mới thay thế init/join tasks để dùng `--cri-socket` đúng

Nếu bạn muốn, mình có thể tạo bộ playbook/role mới này trong repo mà không chỉnh sửa file nào đang có.

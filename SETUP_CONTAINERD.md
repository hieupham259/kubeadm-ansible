# SETUP_CONTAINERD — Kubernetes v1.35.0 (1 master + 2 worker) với containerd

Tài liệu này mô tả các bước **chính xác** để dựng Kubernetes cluster **v1.35.0** gồm **1 master** và **2 worker**, sử dụng **containerd** làm container runtime.

> Ghi chú quan trọng về repo này
>
>- Repo hiện tại là playbook/role kiểu “legacy”:
>  - [roles/commons/pre-install/tasks/pkg.yml](roles/commons/pre-install/tasks/pkg.yml) dùng `apt.kubernetes.io` / `yum.kubernetes.io` (đã bị deprecate/freeze từ 09/2023).
>  - [group_vars/all.yml](group_vars/all.yml) mặc định `kube_version: v1.19.2` và `container_runtime: containerd`.
>  - CNI templates `flannel` và `canal` trong repo dùng các API đã bị xóa ở Kubernetes mới (ví dụ `policy/v1beta1`, `rbac.authorization.k8s.io/v1beta1`, `apiextensions.k8s.io/v1beta1`).
>
> Vì vậy: **Nếu bạn muốn kubeadm/kubelet/kubectl v1.35.0, bạn không nên chạy thẳng `site.yaml` của repo này.**
> Thay vào đó, làm theo các bước dưới (manual) để dựng cluster v1.35.0, và chỉ “tận dụng” repo ở mức inventory/tài liệu, hoặc bạn có thể tạo playbook/role mới (không sửa file cũ) sau khi cluster lên.

---

## A. Chuẩn bị

### A1) Topology
- `master-1` (control-plane): ví dụ `10.0.0.10`
- `worker-1`: ví dụ `10.0.0.11`
- `worker-2`: ví dụ `10.0.0.12`

Yêu cầu:
- Các node có thể ping/route đến nhau.
- Hostname/MAC/product_uuid **không trùng**.
- Mở port kubeadm yêu cầu (6443, 10250, …).

### A2) Chọn OS
Khuyến nghị Ubuntu 22.04/24.04 hoặc distro tương đương. Repo gốc có Vagrant Ubuntu 16.04 nhưng **không phù hợp** cho Kubernetes 1.35.

---

## B. Cấu hình OS chung (chạy trên cả 3 node)

1) Tắt swap:

```bash
sudo swapoff -a
sudo sed -i.bak -r '/\sswap\s/s/^/#/' /etc/fstab
```

2) Bật kernel modules + sysctl cho Kubernetes:

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

## C. Cài containerd (chạy trên cả 3 node)

> Mục tiêu: tạo CRI socket `unix:///var/run/containerd/containerd.sock`.

1) Cài containerd (Ubuntu/Debian):

```bash
sudo apt-get update
sudo apt-get install -y containerd
```

2) Tạo config mặc định và bật `SystemdCgroup=true`:

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null

sudo sed -i -r 's/^\s*SystemdCgroup\s*=\s*false\s*$/            SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable --now containerd
```

3) Kiểm tra socket:

```bash
sudo test -S /var/run/containerd/containerd.sock && echo OK
```

---

## D. Cài kubeadm/kubelet/kubectl v1.35.0 (chạy trên cả 3 node)

Các bước dưới bám theo hướng dẫn chính thống Kubernetes v1.35 (pkgs.k8s.io).

### D1) Ubuntu/Debian

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
```

Cài đúng patch version `1.35.0` (bạn cần chọn đúng chuỗi version mà repo cung cấp):

```bash
apt-cache madison kubeadm | head
apt-cache madison kubelet | head
apt-cache madison kubectl | head

# Ví dụ (version cụ thể phụ thuộc distro):
# sudo apt-get install -y kubelet=1.35.0-1.1 kubeadm=1.35.0-1.1 kubectl=1.35.0-1.1
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

sudo systemctl enable --now kubelet
```

> Lý do phải dùng pkgs.k8s.io: các repo cũ (apt.kubernetes.io/yum.kubernetes.io) đã bị freeze và có thể bị gỡ bỏ bất cứ lúc nào; repo này trong code hiện vẫn dùng repo cũ.

---

## E. Khởi tạo control-plane (chỉ chạy trên master)

1) Kéo images trước (giúp giảm lỗi mạng):

```bash
sudo kubeadm config images pull --kubernetes-version v1.35.0 --cri-socket unix:///var/run/containerd/containerd.sock
```

2) `kubeadm init`:

```bash
sudo kubeadm init \
  --kubernetes-version v1.35.0 \
  --apiserver-advertise-address 10.0.0.10 \
  --service-cidr 10.96.0.0/12 \
  --pod-network-cidr 10.244.0.0/16 \
  --cri-socket unix:///var/run/containerd/containerd.sock
```

3) Setup kubeconfig cho user hiện tại:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

4) Lấy lệnh join chuẩn (an toàn hơn `--discovery-token-unsafe-skip-ca-verification`):

```bash
kubeadm token create --print-join-command
```

---

## F. Join 2 worker (chạy trên worker-1 và worker-2)

Dùng output từ bước E4, thêm `--cri-socket` cho containerd nếu kubeadm không tự detect:

```bash
sudo kubeadm join <MASTER_IP>:6443 --token <...> --discovery-token-ca-cert-hash sha256:<...> \
  --cri-socket unix:///var/run/containerd/containerd.sock
```

---

## G. Cài CNI (bắt buộc) — dùng Calico manifest tương thích

### G1) Vì sao không dùng flannel/canal trong repo
- [roles/cni/templates/flannel.yml.j2](roles/cni/templates/flannel.yml.j2) dùng `policy/v1beta1` và `rbac.authorization.k8s.io/v1beta1` → **không chạy trên Kubernetes mới**.
- [roles/cni/templates/canal.yml.j2](roles/cni/templates/canal.yml.j2) dùng `apiextensions.k8s.io/v1beta1` → **không chạy trên Kubernetes mới**.

### G2) Cài Calico (ví dụ)
Trên master:

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```

> Nếu bạn muốn dùng đúng CNI theo repo này: template Calico trong [roles/cni/templates/calico.yml.j2](roles/cni/templates/calico.yml.j2) rất lớn và khó đảm bảo luôn cập nhật; khuyến nghị dùng manifest chính thức theo phiên bản Calico bạn chọn.

---

## H. Verify

Trên master:

```bash
kubectl get nodes -o wide
kubectl get pods -n kube-system
```

Kỳ vọng:
- 3 node `Ready`
- CoreDNS/Calico pods `Running`

---

## I. Nếu vẫn muốn “Ansible hóa” bằng repo này (không sửa file cũ)

Để repo tự động hóa kubeadm v1.35.0 đúng cách, bạn sẽ cần **tạo thêm file mới** (không sửa file cũ) kiểu:

- Playbook mới: `site-k8s135-containerd.yaml`
- Role mới để cài Kubernetes từ pkgs.k8s.io (thay cho `roles/commons/pre-install` cũ)
- (Tuỳ) role mới để cài containerd chuẩn theo distro

Nếu bạn muốn, mình có thể tạo các playbook/role mới đó trong repo mà **không đụng** vào file hiện có.

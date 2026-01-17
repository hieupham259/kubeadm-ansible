# SETUP_VULTR_UBUNTU2404_CONTAINERD — Dựng K8s v1.35.0 trên Vultr (3 VM)

Mục tiêu: thuê 3 VM Ubuntu 24.04 trên Vultr và dựng cluster Kubernetes **v1.35.0** gồm **1 master** + **2 worker**.

Repo này vốn “legacy” (kube v1.19, repo apt/yum cũ, CNI template cũ). Vì bạn muốn **Ubuntu 24.04 + kubeadm 1.35.0**, mình đã thêm **file mới** (không chỉnh file cũ):

- Inventory riêng: [inventories/vultr/hosts.ini](inventories/vultr/hosts.ini)
- Vars riêng: [inventories/vultr/group_vars/all.yml](inventories/vultr/group_vars/all.yml)
- Playbook mới: [site-ubuntu2404-k8s135-containerd.yaml](site-ubuntu2404-k8s135-containerd.yaml)

Playbook mới dùng:
- `pkgs.k8s.io` để cài kubeadm/kubelet/kubectl v1.35.0
- containerd + CRI socket `unix:///var/run/containerd/containerd.sock`
- Calico manifest chính thức (tránh CNI template cũ không tương thích)

---

## 1) Chuẩn bị trên Vultr

### 1.1 Tạo 3 VM
- OS: Ubuntu 24.04
- Size: tối thiểu 2 vCPU / 2GB RAM cho master (khuyến nghị 2vCPU/4GB)
- Bật IPv4 public
- Add SSH key của bạn (để SSH vào `root`)

### 1.2 (Khuyến nghị) Tạo Vultr VPC
Nếu có thể, tạo 1 VPC và attach cả 3 VM vào cùng VPC.

Lợi ích:
- traffic nội bộ cluster đi private, an toàn hơn
- bạn có thể khóa firewall public “gắt” hơn

Nếu bạn không dùng VPC, cluster vẫn chạy trên public IP, nhưng bạn cần mở port đúng và cẩn thận security.

---

## 2) Chuẩn bị máy chạy Ansible

Trên máy local (Windows) bạn cần:
- Git
- Ansible (khuyến nghị chạy trong WSL2 Ubuntu) **hoặc** một máy Linux làm bastion

> Chạy Ansible native trên Windows thường không “mượt” bằng WSL.

Clone repo:

```bash
git clone <repo-url>
cd kubeadm-ansible
```

---

## 3) Khai báo inventory cho 3 VM

Sửa [inventories/vultr/hosts.ini](inventories/vultr/hosts.ini):

- Thay `CHANGE_ME_MASTER_IP`, `CHANGE_ME_WORKER1_IP`, `CHANGE_ME_WORKER2_IP` bằng IP thật.
- Nếu bạn dùng VPC, bạn có 2 lựa chọn:
  - Dùng public IP cho SSH (ansible_host=public) nhưng vẫn advertise API bằng private IP (xem phần 4)
  - Hoặc SSH thẳng vào private IP nếu bạn có VPN/bastion

Ví dụ nhanh (SSH qua public IP):

```ini
[master]
master-1 ansible_host=203.0.113.10 ansible_user=root

[node]
worker-1 ansible_host=203.0.113.11 ansible_user=root
worker-2 ansible_host=203.0.113.12 ansible_user=root

[kube-cluster:children]
master
node
```

---

## 4) (Tuỳ chọn) Chọn IP advertise cho API server

Mặc định playbook sẽ dùng `ansible_default_ipv4.address` trên master (thường là **public IP**).

Nếu bạn dùng VPC và muốn API server advertise bằng **private IP**, sửa biến sau trong:

- [inventories/vultr/group_vars/all.yml](inventories/vultr/group_vars/all.yml)

```yaml
k8s_apiserver_advertise_address: "<PRIVATE_IP_MASTER>"
```

---

## 5) Chạy playbook dựng cluster

Chạy:

```bash
ansible-playbook -i inventories/vultr/hosts.ini site-ubuntu2404-k8s135-containerd.yaml
```

Playbook sẽ:
- tắt swap
- set sysctl/modules
- cài containerd và bật `SystemdCgroup=true`
- cài kubelet/kubeadm/kubectl **pinned** đúng `1.35.0`
- `kubeadm init` trên master
- `kubeadm join` 2 worker (dùng join command có `--discovery-token-ca-cert-hash`)
- cài Calico

---

## 6) Verify

SSH vào master:

```bash
kubectl --kubeconfig=/etc/kubernetes/admin.conf get nodes -o wide
kubectl --kubeconfig=/etc/kubernetes/admin.conf get pods -n kube-system
```

Kỳ vọng:
- 3 node ở trạng thái `Ready`
- Calico pods `Running`

---

## 7) Ghi chú quan trọng về firewall/security

Nếu bạn chạy trên public IP:
- Chỉ nên mở inbound cần thiết (6443 tới master; 10250/… giữa các node; CNI ports tuỳ plugin)
- Hạn chế IP nguồn (chỉ allow IP office/VPN của bạn) nếu có thể

---

## 8) Nếu bạn muốn Docker option trên Vultr
Bạn đã có tài liệu manual:
- [SETUP_DOCKER.md](SETUP_DOCKER.md)

Nếu bạn muốn mình tạo **playbook Ansible Docker+cri-dockerd** cho Ubuntu 24.04 (cũng chỉ tạo file mới, không sửa file cũ), mình có thể làm tiếp.

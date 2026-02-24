# การติดตั้ง Kubernetes + Docker บน Ubuntu 24.04 LTS
จะติดตั้ง Kubernetes Cluster โดยใช้ Docker Engine เป็น Container Runtime ผ่าน cri-dockerd โดยเน้นความยืดหยุ่นในการเลือกบทบาทของโหนด (Master หรือ Worker)

## 1. การเตรียมระบบ (ทำทุก Node)ตั้งค่า Hostnameตรวจสอบให้แน่ใจว่าแต่ละ Node มีชื่อไม่ซ้ำกัน
ตัวอย่างสำหรับ Master Node หลัก
```bash
sudo hostnamectl set-hostname k8s-master-1
```
ตัวอย่างสำหรับ Master Node เพิ่มเติม
```bash
sudo hostnamectl set-hostname k8s-master-2
```
ตัวอย่างสำหรับ Worker Node
```bash
sudo hostnamectl set-hostname k8s-worker-1
```
ตัวอย่างสำหรับ Worker Node เพิ่มเติม
```bash
sudo hostnamectl set-hostname k8s-worker-2
```
ปิด Swap เนื่องจาก Kubernetes จำเป็นต้องปิด Swap เพื่อการจัดการหน่วยความจำที่แม่นยำ
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
ตั้งค่า Network สำหรับ Bridgingcat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

## 2. ติดตั้ง Docker Engine (ทำทุก Node)
```bash
sudo rm -f /etc/apt/sources.list.d/docker.list

sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
```

## 3. ติดตั้ง cri-dockerd (ทำทุก Node)
```bash
VER=$(curl -s https://api.github.com/repos/Mirantis/cri-dockerd/releases/latest | grep tag_name | cut -d '"' -f 4 | sed 's/v//g')
wget https://github.com/Mirantis/cri-dockerd/releases/download/v${VER}/cri-dockerd-${VER}.amd64.tgz
tar xvf cri-dockerd-${VER}.amd64.tgz
sudo mv cri-dockerd/cri-dockerd /usr/local/bin/

wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service
wget [https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket](https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket)
sudo mv cri-docker.service cri-docker.socket /etc/systemd/system/
sudo sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service

sudo systemctl daemon-reload
sudo systemctl enable --now cri-docker.socket
sudo systemctl enable --now cri-docker.service
```
## 4. ติดตั้ง Kubeadm, Kubelet และ Kubectl (ทำทุก Node)
```bash
sudo rm -f /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
## 5. เริ่มต้น Cluster (เฉพาะ Master Node ตัวแรก)
สำคัญ: หากต้องการเพิ่ม Master ภายหลัง ต้องระบุ IP ของ Master
1. ในช่อง control-plane-endpoint
```bash
sudo kubeadm init \
  --control-plane-endpoint "IP_MASTER_1:6443" \
  --upload-certs \
  --pod-network-cidr=10.244.0.0/16 \
  --cri-socket unix:///var/run/cri-dockerd.sock
```
การตั้งค่า kubectl และแก้ปัญหา TLS Certificateหากพบ Error "tls: failed to verify certificate" ให้รันคำสั่งลบของเก่าแล้วคัดลอกใหม่ดังนี้:
1. ลบไฟล์คอนฟิกเดิมที่ค้างอยู่
```bash
rm -rf $HOME/.kube
```

2. สร้างโฟลเดอร์และคัดลอกไฟล์ใหม่
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
3. เคลียร์ตัวแปรสภาพแวดล้อมที่อาจค้าง
```bash
unset KUBECONFIG
```
6. การ Join โหนดเข้าสู่ Clusterกรณีที่ 1: Join เป็น Worker Node
```bash
sudo kubeadm join IP_MASTER_1:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash> \
    --cri-socket unix:///var/run/cri-dockerd.sock
```
กรณีที่ 2: Join เป็น Master Node (Control Plane)
```bash
sudo kubeadm join IP_MASTER_1:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash> \
    --control-plane --certificate-key <key> \
    --cri-socket unix:///var/run/cri-dockerd.sock
```
8. การยกเลิกและเริ่มใหม่ (Reset Node)
ล้างค่าการตั้งค่าเดิม (บังคับ)
```bash
sudo kubeadm reset -f --cri-socket unix:///var/run/cri-dockerd.sock
```
ล้างข้อมูลเน็ตเวิร์ก
```bash
sudo rm -rf /etc/cni/net.d
sudo rm -rf $HOME/.kube/config
```

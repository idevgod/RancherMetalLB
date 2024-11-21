# คู่มือการติดตั้ง Rancher และการตั้งค่า MetalLB แบบ Manual

## บทนำ
Rancher เป็นแพลตฟอร์มการจัดการ Kubernetes ที่ช่วยให้การจัดการคลัสเตอร์ Kubernetes เป็นเรื่องง่ายและมีประสิทธิภาพ ในการติดตั้ง Rancher บน Kubernetes (K3s) จำเป็นต้องมีการตั้งค่า MetalLB เพื่อจัดการกับบริการแบบ LoadBalancer สำหรับการเข้าถึงจากภายนอก

## ขั้นตอนการติดตั้ง

### 1. ติดตั้ง Helm
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
helm version

helm repo add rancher-alpha https://releases.rancher.com/server-charts/alpha
helm repo add jetstack https://charts.jetstack.io
helm repo update

kubectl create namespace cattle-system

kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/cert-manager.crds.yaml
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true \
  --version v1.11.0
kubectl get pods --namespace cert-manager

helm install rancher rancher-alpha/rancher --devel \
  --namespace cattle-system \
  --set hostname=rancher.cpux.dev \
  --set bootstrapPassword=admin
kubectl -n cattle-system rollout status deploy/rancher
kubectl -n cattle-system get deploy rancher
kubectl get svc -n cattle-system

kubectl expose deployment rancher --name=rancher-lb --port=443 --type=LoadBalancer -n cattle-system
kubectl get svc -n cattle-system

kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/main/config/manifests/metallb-native.yaml
kubectl get nodes -o wide
kubectl -n metallb-system create configmap config --from-literal=address-pools='[
  {
    "name": "default",
    "protocol": "layer2",
    "addresses": ["10.1.240.110-10.1.240.120"]
  }
]'
kubectl get pods -n metallb-system

kubectl describe pods -n metallb-system
kubectl logs -n metallb-system deployment/controller
kubectl logs -n metallb-system deployment/speaker
kubectl get configmap config -n metallb-system -o yaml
ping -c 3 10.1.240.110
ping -c 3 10.1.240.120

kubectl -n cattle-system delete svc rancher-lb
kubectl expose deployment rancher --name=rancher-lb --port=443 --type=LoadBalancer -n cattle-system
kubectl get svc -n cattle-system

kubectl get pods -n metallb-system
kubectl logs -n metallb-system deployment/controller
kubectl logs -n metallb-system deployment/speaker
kubectl describe svc rancher-lb -n cattle-system
kubectl rollout restart deployment/controller -n metallb-system
kubectl rollout restart deployment/speaker -n metallb-system

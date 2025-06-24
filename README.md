## Для всех node (rancher, master & worker)
### Выключаем swap
```
sudo swapoff -a 
```

### Выключаем swap c /etc/fstab
```
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

## Активируют поддержку фильтрации сетевых мостов и включают маршрутизацию IPv4
### Откройте файл /etc/sysctl.conf в текстовом редакторе с правами суперпользователя:
```
sudo nano /etc/sysctl.conf
```
### Добавьте или раскомментируйте следующую строку:
```
net.ipv4.ip_forward = 1
```
Сохраните изменения и закройте редактор.
###  Примените изменения с помощью команды:
```
sudo sysctl -p
```

### Установка необходимых пакетов (если это не ubuntu)
```
sudo apt update && sudo apt install iptables curl -y
```

## Теперь только на rancher сервере
### Входим под рутом
```
sudo -i
```
### Preinstall RKE2
```
mkdir -pv /etc/rancher/rke2
``` 

nano /etc/rancher/rke2/config.yaml  (PS rancher-ts имя хоста)
```
cni:
  - calico
tls-san:
  - rancher-ts
```


### Необходимые алиасы
```
nano ~/.bashrc 
```

```
export CRI_CONFIG_FILE=/var/lib/rancher/rke2/agent/etc/crictl.yaml
export PATH=/var/lib/rancher/rke2/bin:$PATH
export KUBECONFIG='/etc/rancher/rke2/rke2.yaml'

alias crictl='crictl --runtime-endpoint unix:///run/k3s/containerd/containerd.sock'
alias ctr='ctr --address /run/k3s/containerd/containerd.sock --namespace k8s.io'
alias k='kubectl'
```

### Установка RANCHER
```
curl -sfL https://get.rke2.io |sh -
```

```
systemctl enable --now rke2-server
systemctl status rke2-server
```

### Проверка контейнеров (не обязательно)
```
crictl ps
```

###
```
k get nodes
k label node rancher-ts node-role.kubernetes.io/worker=true
```

### Если все готово то устанавливаем HELM
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

### Устанавливаем cert-manager
```
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```

### После того как cert-manager поднимится устанавливаем rancher
```
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm upgrade --install rancher rancher-stable/rancher \
   --namespace cattle-system \
   --set hostname=rancher.micros.uz \
   --set releases=1 \
   --create-namespace
```

## После того как новый k8s раскатается с помощью RANCHER
### Заходим на мастер ноду
```
apt-get install -y apt-transport-https ca-certificates curl gpg software-properties-common

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

apt update
apt install -y kubectl
``` 
```
mkdir -p ~/.kube
cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
chmod 600 ~/.kube/config
```
## --- Шаги для создания кластера с 1 мастер-нодой и 3 воркер-нодам
### После запуска мастер-нода сгенерирует токен для присоединения других нод. Этот токен находится в файле /var/lib/rancher/rke2/server/node-token на мастер-ноде. Скопируйте его (например, K107...).

### На каждой из трёх машин, которые будут воркер-нодам, выполните следующие действия:
```
mkdir -pv /etc/rancher/rke2
```

###
```
server: https://<IP-мастер-ноды>:9345
token: <токен-из-node-token>
```
** Замените <IP-мастер-ноды> на IP-адрес или DNS-имя мастер-ноды (например, rancher-ts или её IP).
** Вставьте <токен-из-node-token>, скопированный с мастер-ноды.

### Установите агент RKE2 (rke2-agent) вместо сервера:
```
curl -sfL https://get.rke2.io | sh -
systemctl enable rke2-agent
systemctl start rke2-agent
```
###  На мастер-ноде проверьте статус узлов
```
/var/lib/rke2/bin/kubectl get nodes
```

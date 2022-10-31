# Rancher HA + RKE + Load Balancer
## Descrição

Processo de instalação, configuração para o rancher versão estável em HA com load balancer e single node ou 3 nodes.

## Tabela de Conteúdo

<!-- TABLE OF CONTENTS -->

## Requisitos:

- [Instalação](#instala%C3%A7%C3%A3o)
- [Instalação Docker](#instala%C3%A7%C3%A3o-docker)
- [Configuração chave ssh](#configura%C3%A7%C3%A3o-chave-ssh)


<!-- ABOUT THE TABLE -->

### Instalação

Pacotes nessários:
  - [kubectl](https://kubernetes.io/docs/tasks/tools/#install-kubectl)
  - [RKE](https://rancher.com/docs/rke/latest/en/installation/)
  - [Docker](https://docs.ranchermanager.rancher.io/v2.5/getting-started/installation-and-upgrade/installation-requirements/install-docker)
  - [Helm](https://helm.sh/docs/intro/install/)

  <!-- TABLE OF CONTENTS -->

## Instalação Docker
Efetuar esse processo em todas as vm
[Ref](https://gist.github.com/guilhermelinhares/9a6fac8b02569fa174e17a3e1de834e3)

```
apt-get update -y
#Init install docker
curl -fsSL https://get.docker.com | bash
sudo usermod -aG docker $USER

#Init install and configure dockerd-rootless-setuptool.sh 
sudo apt-get install -y uidmap
sudo systemctl disable --now docker.service docker.socket
dockerd-rootless-setuptool.sh install
systemctl --user start docker
systemctl --user enable docker
sudo loginctl enable-linger $(whoami)

#Init Install Docker-Composer
sudo apt-get install docker-compose-plugin

#Fix Error:connect EACCES /var/run/docker.sock in VSCODE if after all the steps this error persists run :
#With this command you can assist RW permission for Other user
sudo chmod o+rw /var/run/docker.sock
```

## Configuração Chave SSH

Na vm a qual será o load balancer, gerar uma chave ssh e copiar para os demais hosts

- Gerar chave ssh sem senha
```
ssh-keygen
```
- Copiar chave publica para os demais hosts
```
ssh-copy-id -i ~/.ssh/id_rsa.pub $hostname
```
- Realizar testes de acesso ssh

```
ssh username@hostname
```

### Etapas a serem realizadas na VM que será o Load Balancer:

- [Instalação Kubernets](#instala%C3%A7%C3%A3o-kubernets)
- [Instalação RKE](#instala%C3%A7%C3%A3o-rke)
    - [Inicialização Rke](#inicializa%C3%A7%C3%A3o-rke)
- [Instalação Helm](#instala%C3%A7%C3%A3o-helm)
- [Configuração Rancher File](#configura%C3%A7%C3%A3o-rancher-file)
- [Instalação Rancher](#instala%C3%A7%C3%A3o-rancher)


<!-- ABOUT THE LoadBalancer -->


## Instalação Kubernets

```
sudo curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
sudo chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
kubectl version --client
```

## Instalação Rke

```
sudo curl -LO https://github.com/rancher/rke/releases/download/v1.3.15/rke_linux-amd64
sudo mv rke_linux-amd64 rke
sudo chmod +x rke
sudo mv ./rke /usr/local/bin/rke
rke --version
```

## Instalação Helm
### Ubuntu from apt (Debian/Ubuntu)

[Ref](https://helm.sh/docs/intro/install/#from-script)

### Script
```
sudo curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm help .
```

[Ref](https://helm.sh/docs/intro/install/#from-apt-debianubuntu)
```
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

## Configuração Rancher File

Editar o arquivo *rancher-cluster.yml* conforme as configurações.

- rancher-cluster.yml

#### 3 nodes

```yml
---
nodes:
  - address: hostname
    user: username
    role: [controlplane, worker, etcd]
  - address: hostname
    user: username
    role: [controlplane, worker, etcd]
  - address: hostname
    user: username
    role: [controlplane, worker, etcd]

services:
  etcd:
    snapshot: true
    creation: 6h
    retention: 24h

# Required for external TLS termination with
# ingress-nginx v0.22+
ingress:
  provider: nginx
  options:
    use-forwarded-headers: "true"
---
```
#### Single Node

```yml
---
nodes:
  - address: hostname
    user: username
    role: [controlplane, worker, etcd]

services:
  etcd:
    snapshot: true
    creation: 6h
    retention: 24h

# Required for external TLS termination with
# ingress-nginx v0.22+
ingress:
  provider: nginx
  options:
    use-forwarded-headers: "true"
---
```

### Inicialização Rke

Na pasta a qual foi salvo o arquivo rancher-file.yml executar o seguinte comando

```
rke up --config ./rancher-cluster.yml
```

Após a inicialização do cluster executar os comandos a seguir

```
export KUBECONFIG=$(pwd)/kube_config_rancher-cluster.yml
kubectl get nodes
kubectl get -n ingress-nginx pods
kubectl get pods --all-namespaces
kubectl version
```

### Instalação Rancher

```
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
kubectl create namespace cattle-system
```

### Load Balancer - Nginx

Devemos criar um docker ou uma vm para ser o load balancer para o(s) rancher server(s) para esse tutorial iremos utilizar o load balancer criado em docker.

[Ref](https://docs.ranchermanager.rancher.io/v2.5/how-to-guides/new-user-guides/infrastructure-setup/nginx-load-balancer)

### Configuração arquivo de Configuração Nginx

```
worker_processes 4;
worker_rlimit_nofile 40000;

events {
    worker_connections 8192;
}

stream {
    upstream rancher_servers_http {
        least_conn;
        server <IP_NODE_1>:80 max_fails=3 fail_timeout=5s;
        server <IP_NODE_2>:80 max_fails=3 fail_timeout=5s;
        server <IP_NODE_3>:80 max_fails=3 fail_timeout=5s;
    }
    server {
        listen 80;
        proxy_pass rancher_servers_http;
    }

    upstream rancher_servers_https {
        least_conn;
        server <IP_NODE_1>:443 max_fails=3 fail_timeout=5s;
        server <IP_NODE_2>:443 max_fails=3 fail_timeout=5s;
        server <IP_NODE_3>:443 max_fails=3 fail_timeout=5s;
    }
    server {
        listen     443;
        proxy_pass rancher_servers_https;
    }

}
```

### Instalação
```
docker run -d --restart=unless-stopped \
  -p 80:80 -p 443:443 \
  -v /etc/nginx.conf:/etc/nginx/nginx.conf \
  nginx:1.14
```

## Referências

- [Rancher](https://docs.ranchermanager.rancher.io/v2.5)
- [Helm](https://helm.sh/docs/)

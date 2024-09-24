---
title: Provisionando um cluster Kubernetes no GCP usando Terraform e Kubepray
date: 2024-09-23
spoiler: muita coisa para fazer uma coisa
---
# Provisionando um cluster Kubernetes no GCP usando Terraform e Kubespray

## Introdução e motivação

Já fazia algum tempo que eu estava usando OpenShift e Rancher para gerenciar clusters Kubernetes. Dentre esses clusters, existia um para ferramentas que estava usando RKE (Rancher Kubernetes Engine). Ele estava consumindo bastante recursos, principalmente disco e memória, então resolvi testar o Kubespray para instalação do meu cluster Kubernetes vanilla (padrão).  

## Kubespray

Kubespray é um projeto da CNCF (Cloud Native Computing Foundation), focado em automatizar a instalação de clusters Kubernetes vanilla usando Ansible. É altamente customizável e suporta diversas plataformas de nuvem e on-premise. 

Como ele provisiona um Kubernetes vanilla, ele tende a ser mais leve.

## Pré-requisitos

### Instalação do Terraform 

A instalação do [Terraform CLI](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli) é muito bem documentada no próprio site do fabricante, usando Chocolatey no Windows, Homebrew no Mac e apt no Linux.

### Instalação do gcloud 

Podemos seguir a documentação oficial do Google para instalação do [gcloud CLI](https://cloud.google.com/sdk/docs/install?hl=pt-br). 

Você também precisa de [uma conta](https://console.cloud.google.com/), com o nível gratuito você tem US$ 300 de crédito. 

Após a instalação e criação da sua conta, é necessário que você faça login com o comando: 

```sh
gcloud auth application-default login 
```

### Instalação do Ansible

A instalação do Ansible é bem "amarrada" para o Kubespray, essa é a parte mais complicada, mas vamos lá. 

Você precisa do Python na versão de `3.9` a `3.12`, isso é obrigatório para a versão que eu estou utilizando.  

Após a instalação do `python3.12`, vamos clonar o repositório do projeto Kubespray:

```sh
git clone https://github.com/kubernetes-sigs/kubespray.git
```

Depois vamos instalar o Ansible:

```bash
VENVDIR=kubespray-venv
KUBESPRAYDIR=kubespray
python3.12 -m venv $VENVDIR
source $VENVDIR/bin/activate
cd $KUBESPRAYDIR
pip3.12 install -U -r requirements.txt
pip3.12 install ruamel.yaml
```

## Código Terraform

Escrevi o código usado para subir as máquinas no GCP (Google Cloud Platform) e disponibilizei em https://github.com/fbrunoviana/terraform-kubepray-gpc. Então vamos fazê-lo rodar.

```
git clone https://github.com/fbrunoviana/terraform-kubepray-gpc.git
cd terraform-kubepray-gpc
```

Avalie todas as variáveis, mas principalmente `project_name` e `chave_ssh`. Mude para o seu nome de projeto no GCP e a sua chave pública. Após as mudanças necessárias, suba o cluster.

```
terraform init 
terraform plan -out=plan
terraform apply "plan"
```

E lá vamos nós 🧹 Se tudo der certo, você criou três máquinas no Google Cloud, prontas para o Kubespray.

### Login SSH 

Você receberá como output IPs públicos para acesso, como o exemplo abaixo: 

```yaml
kubernetes_instances = [
  "controller-0: 34.21.26.210",
  "worker-0: 34.16.91.179",
  "worker-1: 35.225.1.235",
]
```

Faça o login em cada uma das máquinas, esse passo é importante para a troca de chaves. Caso tenha limpado a tela, é possível executar o comando `terraform output` e ver novamente os IPs. **Anote cada um dos IPs privados e salve em um bloco de notas.**

```
# volte uma pasta para prosseguir
cd ..
```

## Usando o Kubespray

Certifique-se que está no diretório que clonamos do Kubespray com o `pwd`, feito isso vamos copiar o inventário como manda a documentação.

```
cp -rfp inventory/sample inventory/mycluster
```

Depois criar dinamicamente os IPs:

```
declare -a IPS=($(gcloud compute instances list --filter="tags.items=kubernetes-the-kubespray-way" --format="value(EXTERNAL_IP)"  | tr '\n' ' '))

CONFIG_FILE=inventory/mycluster/hosts.yaml python3.12 contrib/inventory_builder/inventory.py ${IPS[@]}
```

É importante revisar a configuração:

```bash
vim inventory/mycluster/hosts.yaml
```

1. Apague todos os `access_ip`
2. No campo `ip` adicione os IPs privados de cada host.
3. Em `kube_control_plane:` adicione apenas o control-plane.
4. Em `kube_node:` deixe apenas os workers.
5. Em `etcd:` adicione apenas o control-plane.
6. Salve e saia `:wq`

Vamos agora explorar o arquivo que nos fará acessar fora da rede. 

```bash
vim inventory/mycluster/group_vars/k8s_cluster/k8s_cluster.yml
```

1. Altere o valor de `supplementary_addresses_in_ssl_keys` para o IP público do seu control plane, ex: `supplementary_addresses_in_ssl_keys: [34.29.26.210]`
2. Salve e saia `:wq`

Vamos adicionar as métricas no nosso cluster alterando o arquivo:

```
vim inventory/mycluster/group_vars/k8s_cluster/addons.yml
```

1. Mude para `true` o valor de `metrics_server_enabled`
2. Salve e saia `:wq`

### Apertando o botão do spray

Just do it!

```
ansible-playbook -i inventory/mycluster/hosts.yaml -u ubuntu -b -v --private-key=~/.ssh/id_rsa cluster.yml
```

O meu cluster com 6 máquinas demorou cerca de 28 minutos.

### Acessando o cluster K8s 

Volte para o diretório do Terraform e acesse o control plane, vamos precisar dar permissão no arquivo `kubespray-do.conf` e trazê-lo para nossa máquina local:

```
cd terraform-kubepray-gpc
ssh ubuntu@$(terraform output -raw controller_0_ip)

# Na máquina acessada
sudo chown -R ubuntu:ubuntu /etc/kubernetes/admin.conf
exit

# Na sua máquina
scp ubuntu@$(terraform output -raw controller_0_ip):/etc/kubernetes/admin.conf kubespray-do.conf

sed -i 's/127.0.0.1/$(terraform output -raw controller_0_ip)/g' kubespray-do.conf
export KUBECONFIG=$PWD/kubespray-do.conf
```

### Alô som, testando

#### Testando os hosts 

Vamos iniciar os testes agora. 

```
kubectl get nodes 
```

> Output

```
NAME           STATUS   ROLES    AGE   VERSION
controller-0   Ready    master   47m   v1.30.0
controller-1   Ready    master   46m   v1.30.0
controller-2   Ready    master   46m   v1.30.0
worker-0       Ready    <none>   45m   v1.30.0
worker-1       Ready    <none>   45m   v1.30.0
worker-2       Ready    <none>   45m   v1.30.0
```

```
kubectl top nodes
```

> Output

```
NAME           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
controller-0   191m         10%    1956Mi          26%
controller-1   190m         10%    1828Mi          24%
controller-2   182m         10%    1839Mi          24%
worker-0       87m          4%     1265Mi          16%
worker-1       102m         5%     1268Mi          16%
worker-2       108m         5%     1299Mi          17%
```

#### Testando a rede

```shell
kubectl run myshell1 -it --rm --image busybox -- sh
hostname -i
# Rode o myshell2 em outro terminal, exporte novamente o arquivo conf
ping <hostname myshell2>
```

```shell
kubectl run myshell2 -it --rm --image busybox -- sh
hostname -i
ping <hostname myshell1>
```

> Output

```shell
PING 10.233.108.2 (10.233.108.2): 56 data bytes
64 bytes from 10.233.108.2: seq=0 ttl=62 time=2.876 ms
64 bytes from 10.233.108.2: seq=1 ttl=62 time=0.398 ms
64 bytes from 10.233.108.2: seq=2 ttl=62 time=0.378 ms
^C
--- 10.233.108.2 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.378/1.217/2.876 ms
```

#### Deployments

Vamos gerenciar um [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/). Crie um deployment do [nginx](https://nginx.org/en/) web server:

```shell
kubectl create deployment nginx --image=nginx
```

Liste os Deployments:

```shell
kubectl get pods -l app=nginx
```

> Output

```shell
NAME                     READY   STATUS    RESTARTS   AGE
nginx-86c57db685-bmtt8   1/1     Running   0          18s
```

#### Service

**Expor para fora do cluster**

Vamos exportar o nginx usando um [Service](https://kubernetes.io/docs/concepts/services-networking/service/).

Exponha o deployment `nginx` usando um serviço [NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport):

```shell
kubectl expose deployment nginx --port 80 --type NodePort

NODE_PORT=$(kubectl get svc nginx \
  --output=jsonpath='{range .spec.ports[0]}{.nodePort}')
```

Vamos criar uma regra no firewall para liberar a porta:

```sh
gcloud compute firewall-rules create kubernetes-the-kubespray-way-allow-nginx-service \
  --allow=tcp:${NODE_PORT} \
  --network kubernetes-the-kubespray-way
  
EXTERNAL_IP=$(gcloud compute instances describe worker-0 \
  --format 'value(networkInterfaces[0].accessConfigs[0].natIP)')

curl -I http://${EXTERNAL_IP}:${NODE_PORT}
```

### Destruindo 

Então é isso, é bem rápido e fácil de replicar um cluster usando o Kubespray.

Se você fez um laboratório igual ao meu, e se for da sua vontade, é possível destruir com o comando: 

```sh
terraform destroy 
```

Exclua também a regra de firewall que criamos. 

```
gcloud compute firewall-rules delete kubernetes-the-kubespray-way-allow-nginx-service
```

Créditos: 

- https://github.com/kubernetes-sigs/kubespray/blob/master/docs/getting_started/setting-up-your-first-cluster.md
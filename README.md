 - [Objetivo deste documento](http://git.prodam.am.gov.br/dides/kubernetes-docs#objetivo-deste-documento)<br>
 - [O que será construído](http://git.prodam.am.gov.br/dides/kubernetes-docs#o-que-ser%C3%A1-constru%C3%ADdo)<br>
 - [Softwares utilizados e suas versões](http://git.prodam.am.gov.br/dides/kubernetes-docs#softwares-utilizados-e-suas-vers%C3%B5es)<br>
 - [Material de apoio](http://git.prodam.am.gov.br/dides/kubernetes-docs#material-de-apoio)<br>
 - [Instruções de instalação](http://git.prodam.am.gov.br/dides/kubernetes-docs#instru%C3%A7%C3%B5es-de-instala%C3%A7%C3%A3o)<br>
 - [Configuração do Registry Customizado](http://git.prodam.am.gov.br/dides/kubernetes-docs#3-configura%C3%A7%C3%A3o-do-registry-customizado)<br>
 - [Destruição/reseting do cluster](http://git.prodam.am.gov.br/dides/kubernetes-docs#destrui%C3%A7%C3%A3o-do-cluster)<br>

# Objetivo deste documento
Este documento visa documentar a instalação e configuração de um cluster Kubernetes usando duas VM's no Virtualbox. Uma das VM's (denominada kubermaster) será o nó master do cluster, enquanto a outa VM (denominada kubeminion01) representará o nó slave (ou minion) o qual será responsável por executar os PODs do kubernetes.

# O que será construído
- **Master (kubermaster)**: 
Responsável pelo gerenciamento de todo o cluster
- **Slave (kubeminion01)**: 
Nó que hospeda as aplicações

# Softwares utilizados e suas versões
  - Máquina hospedeira: **Ubuntu 16.04**
  - Virtualizador: **Virtualbox 5.1.32**
  - Máquinas virtuais (máquinas que estão rodando no virtualbox): **Ubuntu 16.04**
  - Kubernetes: latest (na época deste doc, **1.9.1**)

# Material de apoio (referencia bibliográfica)
  O processo de configuração deste ambiente foi tomado como base a partir do conhecimento adquirido nos seguintes artigos e documentos:
  - [Building a Kubernetes Cluster in VirtualBox with Ubuntu](https://medium.com/@KevinHoffman/building-a-kubernetes-cluster-in-virtualbox-with-ubuntu-22cd338846dd)
  - [Installing kubeadm](https://kubernetes.io/docs/setup/independent/install-kubeadm/)
  - [Creating a cluster using kubeadm](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/)

# Instruções de instalação

A partir deste ponto, será realizado a configuração de um ambiente virtual utilizando o Virtualbox. O item 1 deste documento prepara as máquinas virtuais (usando virtualbox) para receberem o cluster do kubernetes. 
Caso você não tenha interesse nessa etapa, pode pular para o item 2, onde será realizada a instalação do kubernetes nas máquinas criadas. 
O importante é notar que, caso você não tenha interesse em seguir os passos descritos na seção 1, você deve atentar pelo menos para as configurações de rede que são realizadas nesta seção, para deopis não ter problemas na comunicação dos seus nós e pods no kubernetes.

## 1. Criação e configuração da máquinas virtuais
### 1.1. Criação de nova rede no Virtualbox
  Criei uma rede nova do tipo host-only. Procedimento: 
  <br>
  - Acessar *Arquivo > Preferências > Rede > Redes Exclusivas do Hospedeiro*.
  - Criar rede nova, escolhendo opção host-only
  - Desabilitar DHCP
  - Informar em IPV4 o ip **192.168.99.1**
  

### 1.2. Preparação de máquina virtual base:
  Criei uma máquina no virtualbox com a seguinte configuração: 
  - Nome: kubermaster
  - Sistema operacional: Ubuntu 16.04
  - Rede: NAT (1ª rede) e host-only (2ª rede)
 

  Ao iniciar a máquina virtual, realizei a instalação padrão do Ubuntu.
  Após a instalação, desabilitei o swap através do comando
``` shell
swapoff -a
```
  Também foi necessário acessar o arquivo ``` /etc/fstab ``` e comentar ou remover tudo que se refere a swap.

  Após esse procedimento, instalei o docker (instalação padrão para Ubuntu que pode ser encontrada na própria documentação do docker ou kubernetes, [nesse link](https://kubernetes.io/docs/setup/independent/install-kubeadm/#installing-docker).


### 1.3. Clone das máquinas

 Neste ponto, deve-se realizar um clone das máquinas para a quantidade de nós desejado, obedecendo, no mínimo, as seguintes configurações:
 - Nó master: 2 GB ram e 2 CPU
 - Nó slave (minions do kubernetes): 2 GB RAM e 1 CPU
 

 Para essa configuração de cluster que será montada nesse documento, existe apenas um nó master e N nós slave. Eu criei apenas 1 master e 1 slave, por questões de limitações nos recursos da máquina em que eu estava fazendo os procedimentos.
 
### 1.4. Configurações específicas de cada máquina

 Em cada máquina clonada, deve ser realizado o seguinte procedimento:
 - Configurar o hostname da máquina, através da edição do arquivo ```/etc/hostname```. Por exemplo, o nó master configurei como kubermaster. Já o nó minion, configurei como *kubeminion01*.
 - Configurar o ```/etc/network/interfaces``` da seguinte forma:
  
 
 ``` shell
     # Host-only network
    auto enp0s8
    iface enp0s8 inet static
    address 192.168.99.20
    netmask 255.255.255.0
    network 192.168.99.0
    broadcast 192.168.99.255
  ```
  
  
  Através do comando ```ifconfig -a``` é possível ver as suas interfaces de rede. No meu caso, enp0s8 é o host-only e enp0s3 é a NAT. 
  O campo *address* vai mudar de nó para nó. A configuração que mostrei acima é do meu nó master (ip dele ficou com 192.168.99.20).
 Só o que muda de uma máquina clone para a outra é o *hostname* e o campo *address*
 
### 1.5. Snapshot no Virtualbox

 Até este ponto, a configuração base das máquinas foi realizado, estando elas prontas para receberem o cluster do kubernetes. 
 Neste ponto, cada máquina do cluster deve poder acessar umas as outras através de ssh. 
 No meu caso, precisei instalar o openssh-server em cada nó para que isso fosse possível, mas o mais importante é que elas estejam se enxergando na mesma rede. 
 Logo, é importante fazer este teste de ssh entre os nós para garantir que a rede está ok.
 <br>
 Recomendo criar um snapshot neste ponto pois, caso ocorra algum problema na instalação do cluster do kubernetes, você pode restaurar as máquinas a esse estado e tentar novamente realizar a configuraçaõ do kubernetes.

## 2. Instalação do kubernetes

 A partir de agora será instalado o cluster do kubernetes. Toda a instalação é realizada no nó master, restando um trabalho ínfimo para os nós minions. 
 O procedimento a ser realizado se encontra muito bem documentado [neste link](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/).
 De maneira resumida, a instalação e configuração do cluster kubernetes pode ser compreendida na seguinte sequência de etapas:
 1. [Instalação do kubeadm, kubectl e kubelet](https://kubernetes.io/docs/setup/independent/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl);
 2. Execução do comando ```kubeadm init```, [descrita na seção 2/4 deste link](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/#instructions);
 3. Configuração do kubectl para ser acessível sem usuário root
 4. [Instalação da network das POD's](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/#pod-network);
 5. [Realização do *join* dos nós minios para juntá-los no cluster](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/#44-joining-your-nodes).

#### 2.1 Instalação do kubeadm, kubectl e kubelet
 Para realizar a instalação destas ferramentas, a [documentação oficial do kubernetes](https://kubernetes.io/docs/setup/independent/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl) pode ser seguida na íntegra.
 Neste momento, a instalação dessas ferramentas deve ser feita somente no nó master.
 
#### 2.2 Execução do comando kubeadm init
 O comando deve ser executado da seguinte forma:
 ``` shell
 kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.99.20
 ```
  O ip informado no parâmetro **apiserver-advertise-address** deve ser o do nó master. 
  Já o parâmetro **pod-network-cidr** se refere ao tipo de networkd de POD escolhido (ver a seção 2.4 para mais informações). 
  Portanto, o parâmetro informado aqui está diretamente relacionado ao tipo de rede que foi escolhido (no meu caso, escolhi a rede Flannel).
  <br>
  A saída deste comando informará, no final, algo como:
  
  ``` shell
  You can now join any number of machines by running the following on each node
as root:

  kubeadm join --token <token> <master-ip>:<master-port> --discovery-token-ca-cert-hash sha256:<hash>

```

O comando exibido acima é extremamente importante para permitir a inclusão de nós no cluster. Sem ele, não será possível expandir o cluster com mais nós. É recomendado que esta instrução seja guardada em algum repositório sigiloso da empresa.
  
#### 2.3 Configuração do kubectl acessível como root
 Executar os seguintes comandos:
 ``` shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

 ```
  
#### 2.4 Instalação da network das POD's
 Tal como consta na [documentação oficial](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/#pod-network), é necessário instalar uma network para que as POD's possam comunicar umas com as outras dentro do cluster. Na documentação oficial, existem diversas redes diferentes que podem ser utilizadas, mas tome o cuidado de escolher apenas **uma**. No cluster que criei, escolhi a rede Flannel, pois acredito ser a mais utilizada pela comunidade.
 <br>
 O comando a ser executado para realizar a instalação da rede Flannel é o que segue (executá-lo como root):
 ``` shell
 kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.9.1/Documentation/kube-flannel.yml
 ```
 Também é necessário liberar o tráfego para a rede ipv4 nos iptables definidos pelo kubernetes. Isto pode ser realizado através do comando abaixo:
 ``` shell
 sysctl net.bridge.bridge-nf-call-iptables=1
 ```
 Após executar estes comandos, espere o kube-dns mudar para o estado **Running**. Você pode monitorar este comportamento com o seguinte comando:
 ``` shell
 kubectl get pods --all-namespaces
 ```

#### 2.5 Realização do join dos nós minios para juntá-los no cluster

 Neste ponto, o nó master já está configurado e pronto para ser utilizado, restando tão somente a configuração dos nós slave (ou minions). 
 Isto pode ser realizado através da execução do comando que foi guardado ao término da seção **2.2** desta documentação. 
 <br>Antes de executar este comando, primeiramente é necessário realizar a instalação do kubeadm no nó slave. Isto pode ser realizado através da seguinte sequência de comandos:
 
 
 
``` shell
apt-get update && apt-get install -y apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubeadm
```


 Agora sim é possível executar o comando que foi guardado na saída da seção **2.2**.
 Caso a execução deste comando seja bem sucedida, o nó foi adicionado com sucesso ao cluster e já é possível utilizar o cluster do kubernetes. Para acompanhar se tudo está configurado corretamente, você pode executar o comando ```kubectl get nodes``` para verificar que, neste momento existe um nó master e um nó slave adicionados ao cluster.
 <br>
 O procedimento descrito nesta seção deve ser executado em cada nó slave que se deseje incluir no cluster.
 
## 3. Configuração do registry customizado
Para que seja possível fazer o pull de imagens registradas no registry customizado, primeiramente você deve cadastrar esse registry no cluster do kubernetes. 
Isso é feito através do objeto secret, no kubernetes. Para mais detalhes sobre o funcionamento desse mecanismo, você pode consultar a [documentação oficial do kubernetes](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/).
<br>
Os comandos necessários para registrar o registry customizado:

``` shell
docker login -u admin registry.prodam.am.gov.br
```
 
 
 O comando acima vai solicitar a senha do usuário admin do registry customizado. O comando acima vai gerar um token de autorização que fica armazenado em ```~/.docker/config.json```.
 <br>
 A seguir, o seguinte comando deve ser executado:
 ``` shell
 kubectl create secret docker-registry regsecret --docker-server=registry.prodam.am.gov.br --docker-username=admin --docker-password=<your-pword> --docker-email=<your-email>
 ```
 
 No comando acima, o valor ```<your-pword>``` e ```<your-email>``` devem ser fornecidos para que a chave seja gerada com sucesso.
 <br>
 O registry customizado está configurado com sucesso. Para usá-lo, em cada arquivo yaml você deverá passar uma seção extra que justamente informa o secret que deve ser utilizado, conforme o exemplo a seguir (imagePullSecrets):
 
 ``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-reg
spec:
  containers:
  - name: private-reg-container
    image: <your-private-image>
  imagePullSecrets:
  - name: regsecret
 ```
 
 
## 4. Destruição do cluster
 Caso ocorra alguma falha durante a execução de um dos passos descritos na seção 2 deste documento (instalação do kubernetes), você pode facilmente resetar o cluster ao seu estado original e refazer os passos desde o item 2.1. Para isso, você deve seguir as instruções que constam [nesta seção da documentação do kubernetes](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/#tear-down).

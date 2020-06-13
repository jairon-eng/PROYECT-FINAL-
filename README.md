# DOCUMENTACION FASE ll Ya entregada Y HASTA ABAJO PARTE lll FINAL
# Integrantes del grupo 

- Jose Lopez 
- Silvia Lopez
- Jairon Herrera
- Rony Cifuentes
- Juan Carlos Morales

# [Video de video fase ll](https://drive.google.com/file/d/1QIBtmcpz4nUmnjN4U68KInV0pJU6wXx9/view?usp=sharing)
# [Video Fase Final ](https://umgt-my.sharepoint.com/:v:/g/personal/jherreram6_miumg_edu_gt/ETS1x344IqFMm6Pv0KHysgIB15PdXsx0btgtn3taps4iRw?e=7RcYmB)


# Kubernetes Cluster

Kubernetes es uno de los sistemas de orquestación de contenedores de código abierto más populares. Se utiliza para administrar toda la vida de las aplicaciones en contenedores, incluida la implementación, el escalado, la actualización, etc.

# Diseño de arquitectura

| Nombre del server  | Dirrecion IP |
|---|---|
| master01  | 192.168.100.4  |
| master02  |  192.168.100.5 |

# Prerrequisitos

- Cada servidor tiene al menos 2 cursos de CPU / vCPU, 4 GB de RAM y 10 GB de espacio en disco.
- Todos los servidores deben tener acceso a Internet para descargar paquetes de software.
- El sistema operativo en ellos es CentOS7 con usuario root habilitado.

# Configuracion inicial

### Desactiva el Selinux
- $ setenforce 0
- $ sed -i 's/SELINUX=permissive/SELINUX=disabled/' /etc/sysconfig/selinux

### Desactivar Firewalld
- $ systemctl disable firewalld
- $ systemctl stop firewalld

### Desactivar intercambio
- $ swapoff -a
- $ sed -i 's/^.*swap/#&/' /etc/fstab  

### Habilite el reenvío
- $ iptables -P FORWARD ACCEPT
```
$ cat <  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
vm.swappiness=0
EOF
```
- $ sysctl --system     

Edite el archivo / etc / hosts para que contenga lo siguiente:

- master01 192.168.100.4
- master02 192.168.100.5

### Instalar y configurar Docker

```
$ wget https://download.docker.com/linux/static/stable/x86_64/docker-17.03.2-ce.tgz
$ tar -zxvf docker-17.03.2-ce.tgz
$ cd docker
$ cp * /usr/local/bin
```

Cambie el contenido de /etc/systemd/system/docker.service a lo siguiente

```
[Unit]
Description=Docker Application Container Engine
Documentation=http://docs.docker.io
[Service]
ExecStart=/usr/local/bin/dockerd 
ExecStartPost=/sbin/iptables -I FORWARD -s 0.0.0.0/0 -j ACCEPT
ExecReload=/bin/kill -s HUP $MAINPID
Restart=on-failure
RestartSec=5
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
Delegate=yes
KillMode=process
[Install]
WantedBy=multi-user.target
```
Habilite el servicio Docker y vuelva a cargar la configuración ejecutando los siguientes comandos

- $ systemctl daemon-reload 
- $ systemctl enable docker 
- $ systemctl restart docker

#  Instale kubeadm, kubectl y kubelet

```
$ cat <  /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

$ yum makecache fast && yum install -y kubelet-1.10.0-0 kubeadm-1.10.0-0 kubectl-1.10.0-0

$ cat <  /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=cgroupfs"
EOF

$ systemctl daemon-reload 
$ systemctl enable kubelet 
$ systemctl restart kubelet

```

# Configuración del clúster de Kubernetes

## Generando archivos de configuración maestra
- $ curl -o /usr/local/bin/cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
- $ curl -o /usr/local/bin/cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
- $ chmod +x /usr/local/bin/cfssl*

Cree un archivo de configuración de certificado /opt/ssl/*

- ca-config.json
- ca-csr.json
- etcd-csr.json

- $ cd /opt/ssl/ca-config.json
- $ cfssl gencert -initca ca-csr.json | cfssljson -bare ca
- $ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes etcd-csr.json | cfssljson -bare etcd

## Copiar archivos de configuración a la carpeta de instalación

Para copiar los archivos de configuración generados en el último paso a la carpeta de instalación, debe realizar las siguientes tareas en los 2 nodos maestros.

Crea la carpeta del certificado

* $ mkdir -p /etc/etcd/ssl

Crea el director de datos etcd

* $ mkdir -p /var/lib/etcd

Copie certificados etcd del nodo maestro "Master01"

* $ scp -rp 192.168.100.5:/opt/ssl/*.pem /etc/etcd/ssl/

### Instalar etcd

* $ wget https://github.com/coreos/etcd/releases/download/v3.3.4/etcd-v3.3.4-linux-amd64.tar.gz
* $ tar -zxvf etcd-v3.3.4-linux-amd64.tar.gz
* $ cp etcd-v3.3.4-linux-amd64/etcd* /usr/local/bin/

# Configuración del clúster ETCD

### Master 01

Cree el archivo /etc/systemd/system/etcd.service con el siguiente contenido
```
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/local/bin/etcd \
--name=master01 \
--cert-file=/etc/etcd/ssl/etcd.pem \
--key-file=/etc/etcd/ssl/etcd-key.pem \
--peer-cert-file=/etc/etcd/ssl/etcd.pem \
--peer-key-file=/etc/etcd/ssl/etcd-key.pem \
--trusted-ca-file=/etc/etcd/ssl/ca.pem \
--peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
--initial-advertise-peer-urls=https://192.168.100.04:2380 \
--listen-peer-urls=https://192.168.100.04:2380 \
--listen-client-urls=https://192.168.100.04:2379,http://127.0.0.1:2379 \
--advertise-client-urls=https://192.168.100.04:2379 \
--initial-cluster-token=etcd-cluster-0 \
--initial-cluster=master01=https://192.168.100.04:2380,master02=https://192.168.100.05:2380\
--initial-cluster-state=new \
--data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```
Iniciar servicio etcd

- $ systemctl daemon-reload && systemctl enable etcd && systemctl start etcd

### Master 02

Cree el archivo /etc/systemd/system/etcd.service con el siguiente contenido
```
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/local/bin/etcd \
--name=master01 \
--cert-file=/etc/etcd/ssl/etcd.pem \
--key-file=/etc/etcd/ssl/etcd-key.pem \
--peer-cert-file=/etc/etcd/ssl/etcd.pem \
--peer-key-file=/etc/etcd/ssl/etcd-key.pem \
--trusted-ca-file=/etc/etcd/ssl/ca.pem \
--peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
--initial-advertise-peer-urls=https://192.168.100.05:2380 \
--listen-peer-urls=https://192.168.100.05:2380 \
--listen-client-urls=https://192.168.100.05:2379,http://127.0.0.1:2379 \
--advertise-client-urls=https://192.168.100.05:2379 \
--initial-cluster-token=etcd-cluster-0 \
--initial-cluster=master01=https://192.168.100.04:2380,master02=https://192.168.100.05:2380\
--initial-cluster-state=new \
--data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```
Iniciar servicio etcd

- $ systemctl daemon-reload && systemctl enable etcd && systemctl start etcd


Verifique el estado del clúster ectd

- $ etcdctl --ca-file=/etc/etcd/ssl/ca.pem --cert-file=/etc/etcd/ssl/etcd.pem --key-file=/etc/etcd/ssl/etcd-key.pem cluster-health

## Instalar Keepalived en todos los nodos maestros

### Master01

- $ apt install keepalived

Cree el archivo de configuración keepalived /etc/keepalived/keepalived.conf con el siguiente contenido

```

global_defs {
    router_id LVS_k8s
    }
    
    vrrp_script CheckK8sMaster {
        script "curl -k https://192.168.100.04:6443"
        interval 3
        timeout 9
        fall 2
        rise 2
    }
    
    vrrp_instance VI_1 {
        state BACKUP
        interface eth0
        virtual_router_id 61
        priority 80
        advert_int 1
        mcast_src_ip 192.168.100.04
        nopreempt
        authentication {
            auth_type PASS
            auth_pass KEEPALIVED_AUTH_PASS
        }
        unicast_peer {
            192.168.100.05
        }
        virtual_ipaddress {
            192.168.100.60/24
        }
        track_script {
            CheckK8sMaster
        }
    
    }
```
Comience keepalived

- $ systemctl daemon-reload && systemctl enable keepalived && systemctl restart keepalived

### Master02

- $ apt install keepalived

Cree el archivo de configuración keepalived /etc/keepalived/keepalived.conf con el siguiente contenido

```

global_defs {
    router_id LVS_k8s
    }
    
    vrrp_script CheckK8sMaster {
        script "curl -k https://192.168.100.05:6443"
        interval 3
        timeout 9
        fall 2
        rise 2
    }
    
    vrrp_instance VI_1 {
        state BACKUP
        interface eth0
        virtual_router_id 61
        priority 80
        advert_int 1
        mcast_src_ip 192.168.100.05
        nopreempt
        authentication {
            auth_type PASS
            auth_pass KEEPALIVED_AUTH_PASS
        }
        unicast_peer {
            192.168.100.04
        }
        virtual_ipaddress {
            192.168.100.60/24
        }
        track_script {
            CheckK8sMaster
        }
    
    }
```

Comience keepalived

- $ systemctl daemon-reload && systemctl enable keepalived && systemctl restart keepalived

## Iniciando Master Cluster

### Master01

Cree el archivo kubeadm-config.yaml con el siguiente contenido

``` 
apiVersion: kubeadm.k8s.io/v1alpha1
kind: MasterConfiguration
api:
advertiseAddress: 192.168.100.04
etcd:
endpoints:
- https://192.168.100.04:2379
- https://192.168.100.05:2379
caFile: /etc/etcd/ssl/ca.pem
certFile: /etc/etcd/ssl/etcd.pem
keyFile: /etc/etcd/ssl/etcd-key.pem
networking:
podSubnet: 10.244.0.0/16
apiServerCertSANs:
- 192.168.100.04
- 192.168.100.05
apiServerExtraArgs:
endpoint-reconciler-type: lease

```

Ejecute el siguiente comando para iniciar los servicios de Kubernetes Master

- $ kubeadm init --config kubeadm-config.yaml


Copie archivos pki a todos los demás nodos maestros

- $ scp -rp /etc/kubernetes/pki 192.168.100.04:/etc/kubernetes/
- $ scp -rp /etc/kubernetes/pki 192.168.100.05:/etc/kubernetes/


### Master02

Cree el archivo kubeadm-config.yaml con el siguiente contenido

``` 
apiVersion: kubeadm.k8s.io/v1alpha1
kind: MasterConfiguration
api:
advertiseAddress: 192.168.100.05
etcd:
endpoints:
- https://192.168.100.04:2379
- https://192.168.100.05:2379
caFile: /etc/etcd/ssl/ca.pem
certFile: /etc/etcd/ssl/etcd.pem
keyFile: /etc/etcd/ssl/etcd-key.pem
networking:
podSubnet: 10.244.0.0/16
apiServerCertSANs:
- 192.168.100.04
- 192.168.100.05
apiServerExtraArgs:
endpoint-reconciler-type: lease
```

Ejecute el siguiente comando para iniciar los servicios de Kubernetes Master

- $ kubeadm init --config kubeadm-config.yaml

## Configure Kuberctl en todos los nodos maestros

Ejecute el siguiente comando en todos los nodos maestros

- $ mkdir -p /root/.kube
- $ cp -i /etc/kubernetes/admin.conf /root/.kube/config

## Configurar redes POD

En uno de los maestros ejecute el siguiente comando:

- $ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.10.0/Documentation/kube-flannel.yml


## Instalación de otros componentes / sistemas de soporte
### Instalación del panel de Kubernetes

Cree el archivo dashboard.yaml, archivo adjuntado en repositorio.

Ejecute el siguiente comando en la consola
- $ kubectl create -f dashboard.yaml

## Instalar Helm


La instalación se puede realizar en cualquier nodo maestro

Cree el archivo helm-rbac.yaml como archivo helm RBAC, archivo adjuntado en repositorio.


Ejecute el siguiente comando

- $ kubectl create -f helm-rbac.yaml
- $ wget https://storage.googleapis.com/kubernetes-helm/helm-v2.9.1-linux-amd64.tar.gz
- $ tar -zxvf helm-v2.9.1-linux-amd64.tar.gz
- $ cp linux-amd64/helm /usr/local/bin/

- $ yum install socat -y
- $ helm init --service-account tiller --tiller-namespace kube-system

## Verificación de clúster

Verifique la versión de Kubernetes
- $ kubectl version

Verifique el estado de los componentes de Kubernetes
- $ kubectl get componentstatus

Verifique el estado del nodo

- $ kubectl get node


### Inicie sesión en el panel de control de Kubernetes
Abra https://192.168.100.04:6443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy en su navegador y seleccione la opción Token

Recupere el token del tablero ejecutando el siguiente comando en uno de los nodos maestros.

- $ Dashboard_Secret=`kubectl get secret -n kube-system|grep kubernetes-dashboard-token|awk '{print $1}'`
- $ kubectl describe secret -n kube-system ${Dashboard_Secret} |sed -n '$p'|awk '{print $NF}'

-------------------------------------------------------------------------------------------------------------------

Repositories
Estimated reading time: 5 minutes
Docker Hub repositories allow you share container images with your team, customers, or the Docker community at large.

Docker images are pushed to Docker Hub through the docker push command. A single Docker Hub repository can hold many Docker images (stored as tags).

Creating repositories
To create a repository, sign into Docker Hub, click on Repositories then Create Repository:

Create repo

When creating a new repository:

You can choose to put it in your Docker ID namespace, or in any organization where you are an owner.

The repository name needs to be unique in that namespace, can be two to 255 characters, and can only contain lowercase letters, numbers or - and _.

The description can be up to 100 characters and is used in the search result.

You can link a GitHub or Bitbucket account now, or choose to do it later in the repository settings.

Setting page for creating a repo

After you hit the Create button, you can start using docker push to push images to this repository.

Pushing a Docker container image to Docker Hub
To push an image to Docker Hub, you must first name your local image using your Docker Hub username and the repository name that you created through Docker Hub on the web.

You can add multiple images to a repository by adding a specific :<tag> to them (for example docs/base:testing). If it’s not specified, the tag defaults to latest.

Name your local images using one of these methods:

When you build them, using docker build -t <hub-user>/<repo-name>[:<tag>]

By re-tagging an existing local image docker tag <existing-image> <hub-user>/<repo-name>[:<tag>]

By using docker commit <existing-container> <hub-user>/<repo-name>[:<tag>] to commit changes

Now you can push this repository to the registry designated by its name or tag.

$ docker push <hub-user>/<repo-name>:<tag>
The image is then uploaded and available for use by your teammates and/or the community.

Private repositories
Private repositories let you keep container images private, either to your own account or within an organization or team.

To create a private repository, select Private when creating a repository:

Create Private Repo

You can also make an existing repository private by going to its Settings tab:

Convert Repo to Private

You get one private repository for free with your Docker Hub user account (not usable for organizations you’re a member of). If you need more private repositories for your user account, upgrade your Docker Hub plan from your Billing Information page.

Once the private repository is created, you can push and pull images to and from it using Docker.

Note: You need to be signed in and have access to work with a private repository.

Note: Private repositories are not currently available to search through the top-level search or docker search.

You can designate collaborators and manage their access to a private repository from that repository’s Settings page. You can also toggle the repository’s status between public and private, if you have an available repository slot open. Otherwise, you can upgrade your Docker Hub plan.
#DOCUMENTACION ADICIONAL FINAL DE LOS COMANDOS UTILIZADOS 
#COMANDOS PARA VER LAS IMAGENES DE DOCKER, SELECIONAR UNA SUBIRLA AL REPOSITORIO 
```
docker images
docker commit -m "Example" a426516eef96 jpoou/my-repo:lastest
docker login
docker push jpoou/my-repo:lastest
```

#COMANDOS PARA CREAR UN SECRETO CON LAS CREDENCIALES DEL DOCKER HUB
```
DOCKER_REGISTRY_SERVER=docker.io
DOCKER_USER=user_name
DOCKER_EMAIL=micorreo@hotmail.com
DOCKER_PASSWORD=password
```
#COMANDO QUE CREA UN CODIGO PARA PODER CONECTARSE AL REPOSITORIO PRIVADO DE DOCKER HUB
```
kubectl create secret docker-registry myregistrykey   --docker-server=$DOCKER_REGISTRY_SERVER   --docker-username=$DOCKER_USER   --docker-password=$DOCKER_PASSWORD   --docker-email=$DOCKER_EMAIL
```
#COMANDO PARA VER LOS NODES, EL NOMBRE DEL REPOSITORIO ENTRE OTROS
#CREACION DE ARCHIVO YAML DONDE ESTA LA CONFIGURACION DE NUESTRO REPOSITORIO 
```
kubectl get nodes
vi local.yaml 
kubectl create -f local.yaml 
```
#VER LOS PODS ESTADO, SEGUNDO COMANDO PARA VER LA DESCRIPCION DEL POD 
```
kubectl get pods
kubectl describe pod operativos-final
kubectl get pods
```

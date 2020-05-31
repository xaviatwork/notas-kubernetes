# Kubernetes

Partimos de una imagen base de Debian 10 (Buster) actualizada.

Clonamos la imagen base modificando los identificadores de red de las tarjetas de red en Virtual Box.

## Configuramos el nombre de los nodos

Crearemos un clúster de tres nodos, un *master* y dos *worker nodes*.

| Nodo | IP |
| ---- | -- |
| k-master-0 | 192.168.1.210 |
| k-worker-1 | 192.168.1.201 | 
| k-worker-2 | 192.168.1.202 |

### Modificar el *hostname* de cada nodo

Cambiar el nombre del equipo en los ficheros  `/etc/hosts/` y en `/etc/hostname`

```bash
sudo vi /etc/hosts
sudo vi /etc/hostname
```

### Configuración de red de las VM

Modificamos la configuración de red para fijar una IP estática

```bash
sudo vi /etc/network/interfaces
```

Como buena práctica, realiza una copia del fichero de configuración antes de modificarlo.

```bash
sudo cp /etc/network/interfaces /etc/network/interfaces.bkp
```

Comenta o elimina la línea relativa a la configuración de la tarjeta de red vía DHCP.

La configuración para el nodo `k-master-0` queda:

```ini
~$ cat /etc/network/interfaces
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
allow-hotplug enp0s3
#iface enp0s3 inet dhcp

iface enp0s3 inet static
address 192.168.1.210
netmask 255.255.255.0
broadcast 192.168.1.255
gateway 192.168.1.1
```

En la carpeta `./network-config` tienes el fichero correspondiente a `k-master-0`.

Reinicia el servicio de red:

```bash
sudo systemctl restart networking
```

> Si no funciona, reinicia el nodo.

Repite los pasos indicados para configurar el resto de los nodos, indicando la IP que corresponda.

## SSH-KEY

Se puede conectar a los nodos mediante la clave SSH de la carpeta `./ssh-key`.

## Instalación de kubeadm

[Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/) en la documentación oficial de Kubernetes.

### Prerequisitos

#### Comprobar que las direcciones MAC y los Product UUID son diferentes

Comprobamos que las direcciones MAC de las tarjetas de red de las VMs son diferentes con el comando `ip add show`. También comprobamos que el *Product UUID* es diferente con `sudo cat /sys/class/dmi/id/product_uuid`:

| Nodo | MAC | Product UUID |
| ---- | --- | ------------ |
| k-master-0 | 08:00:27:fb:6b:32 | 0b02e9c4-cfe5-1745-a012-647dd8f84bd0 |
| k--worker-1 | 08:00:27:42:99:a4 | aed3820c-716b-c143-b46d-59898a3206b7 |
| k--worker-2 | 08:00:27:73:ce:54 | 7397e62b-3b05-2b4e-8cfa-0d6b6fa9611d |

### Deshabilitar SWAP

Para deshabilitar la SWAP:

1. Identificamos dónde se encuentra la *swap* `cat /proc/swaps`
1. Deshabilitamos la *swap* `sudo swapoff -a`
1. Editamos el fichero `/etc/fstab` comentando la entrada correspondiente al dispositivo identificado en el primer punto

### Configuración de red para Kubernetes

Permitir tráfico a través de Iptables

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

También hay que validar que el módulo `br_netfilter` está cargado. Lo podemos validar mediante:

```language
$ lsmod | grep br_netfilter
br_netfilter           24576  0
bridge                188416  1 br_netfilter
```

Si no está cargado, lo cargamos explícitamente con `sudo modprobe br_netfilter`.

> Debes realizar la comprobación en todos los nodos del clúster.

### Puertos requeridos

| Protocolo | Dirección     | Rango de puertos | Propósito | Usado por |
| --- | --- | --- | --- | --- | --- |
| TCP | Inbound | 6443*     | Kubernetes | API server | All |
| TCP | Inbound | 2379-2380 | etcd server client API | kube-apiserver, etcd |
| TCP | Inbound | 10250     | Kubelet API | Self, Control plane |
| TCP | Inbound | 10251     | kube-scheduler | Self |
| TCP | Inbound | 10252     | kube-controller-manager | Self |

Worker node(s)

| Protocolo | Dirección     | Rango de puertos | Propósito | Usado por |
| --- | --- | --- | --- | --- | --- |
| TCP | Inbound | 10250 | Kubelet API | Self, Control plane |
| TCP | Inbound | 30000-32767 | NodePort Services† | All |

Los rangos marcados con un asterisco se pueden cambiar, por lo que es necesario asegurarse de que están abiertos.

El plugin de red puede requerir puertos adicionales.

### Instalación del *runtime*

Aunque Kubernetes es compatible con varios *runtimes* de contenedores, instalamos Docker-CE.

Primero, los pre-requisitos para Docker-CE de la [documentación de Docker](https://docs.docker.com/engine/install/debian/) (en la documentación de Kubernetes se indica Ubuntu):

Eliminamos, si las hay, versiones anteriores:

```bash
sudo apt-get remove docker docker-engine docker.io containerd runc
```

Pre-requisitos:

```bash
sudo apt-get update && sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
```

Añadimos la clave GPG oficial de Docker

```bash
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
```

Añadimos el repositorio de Docker:

```bash
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/debian \
   $(lsb_release -cs) \
   stable"
```

Instalamos Docker-CE

```bash
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

Tras un rato, puedes comprobar que Docker-CE se ha instalado correctamente con:

```bash
docker version
Client: Docker Engine - Community
 Version:           19.03.10
 API version:       1.40
 Go version:        go1.13.10
 Git commit:        9424aeaee9
 Built:             Thu May 28 22:17:05 2020
 OS/Arch:           linux/amd64
 Experimental:      false
Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Get http://%2Fvar%2Frun%2Fdocker.sock/v1.40/version: dial unix /var/run/docker.sock: connect: permission denied
```

### Instalación de kubeadm, kubelet y kubectl

kubeadm no instala ni *kubelet* ni *kubectl*, por lo que debes asegurarte de instalarlo manualmente. También debes tener en cuenta que deben estar en la misma versión para que no haya problemas.

En [Install and configure kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) se detalla cómo instalar y configurar *kubectl*.

Instalamos los tres componentes:

```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

El *kubelet* está reiniciándose constantemente, ya que está a la espera de recibir instrucciones de *kubeadm*.

> Hasta ahora todas las acciones son comunes en los todos los nodos del clúster, sin tener en cuenta el rol que juegan en él. Es un buen momento para hacer un *snapshot*. También validamos que cada nodo tiene dos vCPUs asignadas.

## Instalación del *control plane* con *kubeadm*

Una vez instaladas todas las piezas, es el momento de usar *kubeadm* para crear un *control plane*.

Aunque tengo planeado actualizar el clúster más adelante para configurar HA para el *control plane*, en estos momentos no tengo un *load balancer*, por lo que no puedo realizar la configuración recomendada de indicar el `--control-plane-endpoint`.

Como indica la [documentación](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#tear-down), *kubeadm* no soporta pasar de un clúster mono-nodo inicializado sin especificar `--control-plane-endpoint` a un cluster con HA. Como este parámetro soporta tanto direcciones IP como nombres DNS, es posible definir un nombre DNS apuntando a la IP del único nodo del clúster y pasar este nombre DNS como valor de `--control-plane-endpoint`; de esta forma podemos modificar más adelante la entrada DNS para apuntar a un balanceador (sin tener que modificar la configuración del clúster).

Creamos una "entrada DNS" en el fichero `/etc/hosts` del nodo `k-master-0` con apuntando su IP al nombre `cluster-endpoint`.

Lanzamos **como `root`** la inicialización del *control plane* con el comando:

```bash
sudo kubeadm init --control-plane-endpoint=cluster-endpoint
```

Si lanzamos el comando sin `sudo`, fallan las comprobaciones previas a la inicialización del proceso:

```bash
$ kubeadm init --control-plane-endpoint=cluster-endpoint
W0530 23:45:02.132143    1421 configset.go:202] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
[init] Using Kubernetes version: v1.18.3
[preflight] Running pre-flight checks
error execution phase preflight: [preflight] Some fatal errors occurred:
        [ERROR IsPrivilegedUser]: user is not running as root
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
To see the stack trace of this error execute with --v=5 or higher
```

Tras lanzar el comando como `root`, *kubeadm* descarga las imágenes necesarias para desplegar el *control plane*. Es posible agilizar este proceso descargando las imágenes previamente con `kubeadm config images pull`.

Tras un par de minutos, la instalación concluye:

```bash
$ sudo  kubeadm init --control-plane-endpoint=cluster-endpoint
W0530 23:47:06.244099    1516 configset.go:202] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
[init] Using Kubernetes version: v1.18.3
[preflight] Running pre-flight checks
        [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [k-master-0 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local cluster-endpoint] and IPs [10.96.0.1 192.168.1.210]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [k-master-0 localhost] and IPs [192.168.1.210 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [k-master-0 localhost] and IPs [192.168.1.210 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
W0530 23:49:39.075187    1516 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[control-plane] Creating static Pod manifest for "kube-scheduler"
W0530 23:49:39.077261    1516 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 23.006612 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.18" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node k-master-0 as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node k-master-0 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: 057l8q.y8ybcouzoql7c30l
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join cluster-endpoint:6443 --token 057l8q.y8ybcouzoql7c30l \
    --discovery-token-ca-cert-hash sha256:540a59b4a70db7478d0019822168551df660ec7f0da2fd8f424bba64816bf92e \
    --control-plane

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join cluster-endpoint:6443 --token 057l8q.y8ybcouzoql7c30l \
    --discovery-token-ca-cert-hash sha256:540a59b4a70db7478d0019822168551df660ec7f0da2fd8f424bba64816bf92e
```

Seguimos las instrucciones de la salida de la inicialización del clúster para configurar *kubectl*:

```bash
mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

El siguiente paso es la instalación del plugin CNI para la red de los *pods*.

### Instalación de la red para los *pods*

> Si no se instalan un plugin de red que proporciona la red para los *pods*, el servicio de Cluster DNS (Core DNS) no arranca.

En estos momentos Calico es el único plugin CNI que se prueba de forma extensiva con *kubeadm*.

Para instalar Calico:

```bash
$ kubectl apply -f https://docs.projectcalico.org/v3.14/manifests/calico.yaml
configmap/calico-config created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org created
clusterrole.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrole.rbac.authorization.k8s.io/calico-node created
clusterrolebinding.rbac.authorization.k8s.io/calico-node created
daemonset.apps/calico-node created
serviceaccount/calico-node created
deployment.apps/calico-kube-controllers created
serviceaccount/calico-kube-controllers created
```

Tras desplegar el plugin CNI de Calico, podemos observar la creación de los *pods* necesarios mediante:

```bash
kubectl get pods --all-namespaces -w
```

Tras dejar pasar un tiempo prudencial, observamos que todos los *pods* han arrancado correctamente:

```bash
$ kubectl get pods --all-namespaces -w
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-76d4774d89-rz578   1/1     Running   0          3m10s
kube-system   calico-node-bsspz                          1/1     Running   0          3m10s
kube-system   coredns-66bff467f8-4rq8t                   1/1     Running   0          5m43s
kube-system   coredns-66bff467f8-kn7gt                   1/1     Running   0          5m43s
kube-system   etcd-k-master-0                            1/1     Running   0          5m44s
kube-system   kube-apiserver-k-master-0                  1/1     Running   0          5m44s
kube-system   kube-controller-manager-k-master-0         1/1     Running   0          5m44s
kube-system   kube-proxy-5df4v                           1/1     Running   0          5m43s
kube-system   kube-scheduler-k-master-0                  1/1     Running   0          5m44s
```

## Añadir nodos worker al clúster

> Para que el comando funcione es necesario que el nodo *worker* pueda resolver el nombre `cluster-endpoint`. Lo añadimos al fichero `/etc/hosts` de los nodos *workers*.

Desde el nodo `k-worker-1` lanzamos el comando indicado en la salida de *kubeadm* para añadir el nodo al clúster:

```bash
$ sudo kubeadm join cluster-endpoint:6443 --token 057l8q.y8ybcouzoql7c30l     --discovery-token-ca-cert-hash sha256:540a59b4a70db7478d0019822168551df660ec7f0da2fd8f424bba64816bf92e
W0531 00:03:41.751723    1978 join.go:346] [preflight] WARNING: JoinControlPane.controlPlane settings will be ignored when control-plane flag is not set.
[preflight] Running pre-flight checks
        [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.18" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

En el nodo `k-master-0` podemos observar cómo el nodo aparece en el clúster (aunque inicialmente en estado *NotReady*) usando el comando:

```bash
$ kubectl get nodes
NAME         STATUS   ROLES    AGE    VERSION
k-master-0   Ready    master   15m    v1.18.3
k-worker-1   Ready    <none>   117s   v1.18.3
```

Repetimos en el segundo nodo *worker*.

```bash
$ kubectl get nodes
NAME         STATUS   ROLES    AGE     VERSION
k-master-0   Ready    master   18m     v1.18.3
k-worker-1   Ready    <none>   4m34s   v1.18.3
k-worker-2   Ready    <none>   92s     v1.18.3
```

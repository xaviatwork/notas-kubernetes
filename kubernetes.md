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

### Configuración de red

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

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

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

### Configuración de red

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
| TCP | Inbound | 6443*	    | Kubernetes | API server | All | 
| TCP | Inbound | 2379-2380	| etcd server client API	| kube-apiserver, etcd |
| TCP | Inbound | 10250	    | Kubelet API | Self, Control plane |
| TCP | Inbound | 10251	    | kube-scheduler | Self | 
| TCP | Inbound | 10252	    | kube-controller-manager | Self |

Worker node(s)

| Protocolo | Dirección     | Rango de puertos | Propósito | Usado por |
| --- | --- | --- | --- | --- | --- |
| TCP | Inbound | 10250	| Kubelet API | Self, Control plane |
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


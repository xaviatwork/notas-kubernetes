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


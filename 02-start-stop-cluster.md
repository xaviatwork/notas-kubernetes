# Procedimiento para arrancar/parar el clúster

Para parar el clúster, primero paramos los nodos *worker* desde el sistema operativo (o lanzando la señal de apagado desde el hypervisor).

```bash
 ssh operador@k-worker-1 -t sudo shutdown now
 ssh operador@k-worker-2 -t sudo shutdown now
 ssh operador@k-master-0 -t sudo shutdown now
```

Para arrancar el clúster, primero inicia los nodos *worker* desde el hypervisor y después el nodo *master*.

## Script para Virtual Box en Windows

VirtualBox permite gestionar las máquinas virtuales desde la línea de comando mediante el comando `VBoxManage` (ubicado en `C:\Program Files\Oracle\VirtualBox`).

Puedes obtener un listado de las máquinas virtuales con:

```bash
PS> C:\Program Files\Oracle\VirtualBox\vboxmanage list vms
.\VBoxManage.exe list vms
"k-master-0" {c4e9020b-e5cf-4517-a012-647dd8f84bd0}
"k-worker-1" {0c82d3ae-6b71-43c1-b46d-59898a3206b7}
"k-worker-2" {2be69773-053b-4e2b-8cfa-0d6b6fa9611d}
```

Para arrancar una VM, puedes indicar el UUID o el nombre, en nuestro caso:

> Recuerdo que debes dejar pasar un cierto intervalo de tiempo entre el arranque de los *workers* y el *master*.

```bash
 "C:\Program Files\Oracle\VirtualBoxVBoxManage.exe startvm k-worker-2 --type headless"
 "C:\Program Files\Oracle\VirtualBoxVBoxManage.exe startvm k-worker-1 --type headless"
 "C:\Program Files\Oracle\VirtualBoxVBoxManage.exe startvm k-master-0 --type headless"
```

Para parar el clúster:

```bash
 "C:\Program Files\Oracle\VirtualBoxVBoxManage.exe controlvm k-worker-2 acpipowerbutton"
 "C:\Program Files\Oracle\VirtualBoxVBoxManage.exe controlvm k-worker-1 acpipowerbutton"
 "C:\Program Files\Oracle\VirtualBoxVBoxManage.exe controlvm k-master-0 acpipowerbutton"
```

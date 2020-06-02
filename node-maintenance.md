# Mantenimiento de los nodos

Referencia: [Safely Drain a Node while Respecting the PodDisruptionBudget](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/).

Para elimianar de manera segura los *pods* de un node de Kubernetes, usamos el comando `kubectl drain`:

```bash
$ kubectl get nodes
$ kubectl drain k-worker-3
node/k-worker-3 already cordoned
error: unable to drain node "k-worker-3", aborting command...

There are pending nodes to be drained:
 k-worker-3
error: cannot delete DaemonSet-managed Pods (use --ignore-daemonsets to ignore): kube-system/calico-node-ztm6t, kube-system/kube-proxy-bw9dz
```

El comando `kubectl drain` ignao algunos *pods* de ssitema que no pueden eliminarse (como los de red). Al lanzar el comando, Kubernetes lanza la señal de apagado a los *pods*, de manera que puedan apagarse correctamente. Una vez el nodo se ha drenado, lo podemos apagar.

Si obtenemos la lista de nodos en el clúster, observamos que el nodo está en estado *Ready*, pero que tiene el *scheduling disabled* (es decir, no se pueden desplegar nuevos *pods* en él).

```bash
NAME         STATUS                     ROLES    AGE     VERSION
k-master-0   Ready                      master   2d22h   v1.18.3
k-worker-1   Ready                      <none>   2d21h   v1.18.3
k-worker-2   Ready                      <none>   2d21h   v1.18.3
k-worker-3   Ready,SchedulingDisabled   <none>   2d4h    v1.18.3
```

Una vez finalizada la tarea de mantenimiento, usamos el comando `kubectl uncordon` para permitir que se vuelvan a desplegar *pods* en el nodo.

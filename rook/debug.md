$ kubectl get svc -A
NAMESPACE     NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
default       kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP                  13h
kube-system   kube-dns     ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   13h


$ kubectl get pods -n rook-ceph
NAME                                  READY   STATUS             RESTARTS   AGE
rook-ceph-operator-757d6db48d-9gdzz   0/1     CrashLoopBackOff   7          20m

$ kubectl logs -n rook-ceph rook-ceph-operator-757d6db48d-9gdzz
2020-06-06 11:07:37.563872 I | rookcmd: starting Rook v1.3.4 with arguments '/usr/local/bin/rook ceph operator'
2020-06-06 11:07:37.564033 I | rookcmd: flag values: --add_dir_header=false, --alsologtostderr=false, --csi-cephfs-plugin-template-path=/etc/ceph-csi/cephfs/csi-cephfsplugin.yaml, --csi-cephfs-provisioner-dep-template-path=/etc/ceph-csi/cephfs/csi-cephfsplugin-provisioner-dep.yaml, --csi-cephfs-provisioner-sts-template-path=/etc/ceph-csi/cephfs/csi-cephfsplugin-provisioner-sts.yaml, --csi-rbd-plugin-template-path=/etc/ceph-csi/rbd/csi-rbdplugin.yaml, --csi-rbd-provisioner-dep-template-path=/etc/ceph-csi/rbd/csi-rbdplugin-provisioner-dep.yaml, --csi-rbd-provisioner-sts-template-path=/etc/ceph-csi/rbd/csi-rbdplugin-provisioner-sts.yaml, --enable-discovery-daemon=true, --enable-flex-driver=false, --enable-machine-disruption-budget=false, --help=false, --kubeconfig=, --log-flush-frequency=5s, --log-level=INFO, --log_backtrace_at=:0, --log_dir=, --log_file=, --log_file_max_size=1800, --logtostderr=true, --master=, --mon-healthcheck-interval=45s, --mon-out-timeout=10m0s, --operator-image=, --service-account=, --skip_headers=false, --skip_log_headers=false, --stderrthreshold=2, --v=0, --vmodule=
2020-06-06 11:07:37.564056 I | cephcmd: starting operator
failed to get pod: Get https://10.96.0.1:443/api/v1/namespaces/rook-ceph/pods/rook-ceph-operator-757d6db48d-9gdzz: dial tcp 10.96.0.1:443: i/o timeout

$ kubectl describe configmap/coredns -n kube-system
Name:         coredns
Namespace:    kube-system
Labels:       <none>
Annotations:  <none>

Data
====
Corefile:
----
.:53 {
    errors
    health {
       lameduck 5s
    }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
       ttl 30
    }
    prometheus :9153
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
    loadbalance
}

Events:  <none>
operador@k-manager:~/rook-deploy$ cat /etc/resolv.conf
domain Home
search Home
nameserver 212.231.6.7
nameserver 46.6.113.34

Creo que el problema puede estar aquí: como CoreDNS hace un `forward` al DNS que obtiene de `/etc/resolv.conf` y éste no contiene la IP del clúster, no encuentra el servicio.

Esto me lleva de nuevo a que tengo que montar un DNS en el laboratorio y configurar los nombres de los nodos del clúster ahí.


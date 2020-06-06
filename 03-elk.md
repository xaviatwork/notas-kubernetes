# ELK: Elasticseatch, Logstash y Kibana

Voy a usar las instrucciones de Digital Ocean [How To Set Up an Elasticsearch, Fluentd and Kibana (EFK) Logging Stack on Kubernetes](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-elasticsearch-fluentd-and-kibana-efk-logging-stack-on-kubernetes)

## Crear un *namespace*

Creamos un esapcio de nombres donde desplegaremos los *pods*:

```yaml
kind: Namespace
apiVersion: v1
metadata:
  name: kube-logging
```

Aplicamos el fichero para crear el *namespace*:

```bash
$ kubectl create -f elasticsearch_namespace_kube-logging.yaml
namespace/kube-logging created
```

Validamos que se ha creado mediante:

```bash
PS > .\kubectl.exe get ns
NAME              STATUS   AGE
default           Active   17h
kube-logging      Active   8s
kube-node-lease   Active   17h
kube-public       Active   17h
kube-system       Active   17h
```

## Crear el *statefulset* para ElasticSearch

Una vez creado el *namespace* empezamos a desplegar los diferentes componentes que necesitamos.

Empezamos desplegando un cluster de tres nodos de ElasticSearch. Usamos tres nodos para evitar el problema del *split brain* que se puede dar en un clúster con múltiples nodos.

### Crear el servicio *headless*

Creamos el fichero `elasticsearch_service.yaml`:

```yaml
kind: Service
apiVersion: v1
metadata:
  name: elasticsearch
  namespace: kube-logging
  labels:
    app: elasticsearch
spec:
  selector:
    app: elasticsearch
  clusterIP: None
  ports:
    - port: 9200
      name: rest
    - port: 9300
      name: inter-node
```

Definimos un servicio llamado *elasticsearch* en el espacio de nombres `kube-logging` y le aplicamos la etiqueta *app:elasticsearch*. Definimos el *selector* como `app: elasticsearch`. Como especificamos `ClusterIP: None` lo convertimos en *headless* y a continuación exponemos los puertos 9200 para la API REST y el 9300 para la comunicación entre nodos.

Creamos el servicio mediante:

```bash
$ kubectl create -f service_elasticsearch.yaml
service/elasticsearch created
```

Validamos que se ha creado mediante:

```bash
$ kubectl get services -n kube-logging
NAME            TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)             AGE
elasticsearch   ClusterIP   None         <none>        9200/TCP,9300/TCP   63s
```

### Crear el *StatefulSet*

Un *StatefulSet* en Kubernetes permite asignar una identidad estable a los *pods*, así como almacenamiento que sobrevida a los reinicios de los *pods*.

Creamos un fichero para definir el *statefulSet*: `elasticsearch_statefulset.yml`.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: es-cluster
  namespace: kube-logging
spec:
  serviceName: elasticsearch
  replicas: 3
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
```

En este fichero definimos un *StatefulSet* llamado `es-cluster` en el espacio de nombres `kube-logging`. Le asociamos el servicio creado anteriormente llamado  `elasticsearch` a través de `.spec.serviceName`. Esto asegura que cada *pod* en el *StatefulSet* será accesible con el nombre `es-cluster-[0,1,2].elasticserch.kube-logging.svc.cluster.local` (porque hemos especificado tres réplicas).

...

> No tengo una *storage class* en el clúster, por lo que no podré desplegar el *StatefulSet* (que algún tipo de almacenamiento).

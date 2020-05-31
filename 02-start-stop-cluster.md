# Procedimiento para arrancar/parar el clúster

Para parar el clúster, primero paramos los nodos *worker* desde el sistema operativo (o lanzando la señal de apagado desde el hypervisor).

```bash
 ssh operador@k-worker-1 -t sudo shutdown now
 ssh operador@k-worker-2 -t sudo shutdown now
 ssh operador@k-master-0 -t sudo shutdown now
```

Para arrancar el clúster, primero inicia los nodos *worker* desde el hypervisor y después el nodo *master*.

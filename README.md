# POC Deploy Wordpress con Autoscaling y HA en Kubernetes

#### Introduccion

Prueba de concepto sobre una arquitectura de microservicios corriendo en un cluster de Kubernetes, contemplando escalabilidad y alta disponibilidad. 
Para esto se usara Wordpress como frontend y backend con dos nodos inicialmente, y una base de datos MySQL.

#### Prerequisitos

Crear un ambientes kubernetes corriendo con Minikube.

#### Ambiente

- Minikube v0.26.1
- Kubernetes v1.10
- Ubuntu v16.04/amd64

#### Objetivos

1. Crear volumenes persistentes 
2. Crear un "secret" para proteger datos sensibles
3. Crear y deployar una base de datos Mysql
4. Crear y deployar Wordpress
5. Testear la aplicacion
6. Testear escalabilidad horizontal
7. Configurar Autoscaling Webserver
8. Testear Alta Disponibilidad

------



#### 1. Crear volumenes persistentes 

Para guardar la informacion y que persista al ciclo de vida de un pod crearemos volumenes persistentes:

```
$ kubectl create -f volumes.yml
```



#### 2. Crear un "secret" para proteger datos sensibles

```
$ kubectl create secret generic mysql-pass --from-literal=password=XXXXXXXX
```



#### 3. Crear y deployar una base de datos MySQL 

```
$ kubectl create -f mysql.yml
```



#### 4. Crear y deployar Wordpress

```
$ kubectl create -f wordpress.yml
```



#### 5. Testear la aplicacion

Antes de seguir podemos verificar los pods, los deployments y los servicios

```
$ kubectl get pods,deploy,svc
NAME                              READY     STATUS    RESTARTS   AGE
wordpress-7467f54f7d-rz99q        1/1       Running   0          10m
wordpress-7467f54f7d-x8f5f        1/1       Running   0          10m
wordpress-mysql-bcc89f687-56v9h   1/1       Running   0          10m
wordpress-mysql-bcc89f687-t4vf6   1/1       Running   0          10m

NAME              DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
wordpress         2         2         2            2           10m
wordpress-mysql   2         2         2            2           10m

NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes        ClusterIP   10.96.0.1       <none>        443/TCP        1h
wordpress         NodePort    10.111.70.39    <none>        80:30010/TCP   10m
wordpress-mysql   ClusterIP   10.104.80.110   <none>        3306/TCP       10m
```

Tambien podemos verificar los Persistent Volume Claims

```
$ kubectl get pvc
NAME             STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mysql-pv-claim   Bound     pvc-92e0738c-46fa-11e8-9e6b-080027d297e2   1Gi        RWO            standard       12m
wp-pv-claim      Bound     pvc-960c4a8a-46fa-11e8-9e6b-080027d297e2   1Gi        RWO            standard       12m
```

Para testear la aplicacion tendremos que entrar con el browser a la url que nos indique minikube

```
$ minikube service wordpress --url
```

Frontend: http://$urlminikube/
Backend: http://$urlminikube/wp-admin

###### Para acceder al backend primero hay que completar la instalacion del WP.



#### 6. Testear escalabilidad horizontal 

Podremos aumentar la cantidad de replicas de forma manual para repartirar la carga.

```
$ kubectl scale deployments/wordpress --replicas=4
```

Luego podremos verificar el estado de los deployments para ver como aumentaron las replicas a 4

```
$ kubectl get deploy
NAME              DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
wordpress         4         4         4            2           14m
wordpress-mysql   2         2         2            1           14m
```



#### 7. Configurar autoscaling

Esta configurado el autoscaling (Horizontal Pod Autoscaler - HPA) para que minimo haya 2 replicas y maximo 10, y crezca cuando llegue a un %20 de consumo de CPU (valor bajo para testear mas rapidamente), tambien se puede verificar el estado actual del HPA

```
$ kubectl create -f wordpress-hpa.yml
$ kubectl get hpa
NAME        REFERENCE              TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
wordpress   Deployment/wordpress   <unknown>/20%   2         10        4          1m
```

###### Por un bug en Minikube desde la version 1.8 de Kubernetes no trae las metricas de hardware y por lo tanto no puede utilizar HPA, funciona correctamente en un entorno corriendo en GCE. 

[Github]: https://github.com/kubernetes/kubernetes/issues/57673	"Hpa problem on k8s"


Podemos generar carga en los pods para verificar como escala automaticamente.
Corrrera 100000 request con una concurrencia de 10 (necesita el paquete apache2-utils)

```
$ ab -n 10000 -c 10 http://ip:port/
```



#### 8. Testear Alta Disponibilidad

Kubernetes por default brinda alta disponibilidad, esto lo logra implementando componentes de auto regeneracion. Esto se puede comprobar eliminando uno de los pods.

```
$ kubectl delete pod wordpress
```
Si vemos el nuevo estado de los podes veremos como automaticamente se genera uno que reemplaza al anterior

```
$ kubectl get pods
NAME                              READY     STATUS              RESTARTS   AGE
wordpress-7467f54f7d-5gjxb        0/1       Terminating         0          17m
wordpress-7467f54f7d-p7qp2        0/1       ContainerCreating   0          1s
```

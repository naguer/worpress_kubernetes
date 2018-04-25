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

1. Crear un "secret" para proteger datos sensibles
2. Crear y deployar una base de datos Mysql
3. Crear y deployar Wordpress
4. Testear la aplicacion
5. Testear escalabilidad horizontal
6. Configurar Autoscaling Webserver
7. Testear Alta Disponibilidad
8. Extra: Levantar entorno en Google Compute Engine

------



#### 1. Crear un "secret" para proteger datos sensibles

```
$ kubectl create secret generic mysql-pass --from-literal=password=XXXXXXXX
```



#### 2. Crear y deployar una base de datos MySQL 

```
$ kubectl create -f mysql.yml
```



#### 3. Crear y deployar Wordpress

```
$ kubectl create -f wordpress.yml
```



#### 4. Testear la aplicacion

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



#### 5. Testear escalabilidad horizontal 

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



#### 6. Configurar autoscaling

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



#### 7. Testear Alta Disponibilidad

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



####  8. Extra: Levantar entorno en Google Compute Engine

Para levantar el entorno en Google Compute Engine hay que tener una cuenta Google y solicitar el free tier (u$s 300 por 12 meses) en https://cloud.google.com/free/

Luego hay que instalar el SDK de Gcloud en el OS https://cloud.google.com/sdk/downloads

Una vez que ya esta instalado Gcloud y solicitado el freetier se puede comenzar a levantar el entorno.
Nos pedira que ingresemos la direccion de mail que tiene asociado el Free Tier, y luego en el browser que validemos el acceso.

```
gcloud init
```

Creamos el nombre del proyecto que vamos a utilizar y configuramos los valores por defecto

```
$ gcloud projects create [PROJECT_ID]
$ gcloud config set project [PROJECT_ID]
$ gcloud config set compute/zone us-central1-c
$ gcloud config set compute/region us-central1
$ gcloud config list 
```

Configuramos variables de entorno (Linux)

```
$ export CLUSTER_NAME="wordpress"
$ export ZONE="us-central1-c"
$ export REGION="us-central1"
```

Creamos el cluster de kubernetes, esto puede tardar varios minutos.

```
$ gcloud container clusters create wordpress-cluster --num-nodes 3
```

Ahora configuramos el cluster como el default.

```
$ gcloud config set container/cluster wordpress-cluster
```

Por ultimo, para levantar el entorno vamos al paso 1) y seguimos los mismos pasos que hariamos en Minikube
La unica diferencia sera que para ver la ip publica que tendremos que acceder para ver nuestra aplicacion, sera la ip que esta definida como EXTERNAL-IP, aparecera asignada cuando el servicio este disponible.

```
kubectl get svc wordpress
NAME        TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
wordpress   LoadBalancer   10.51.245.83   35.193.199.130   80:32017/TCP   2m
```

Si queremos limpiar todo rapidamente se puede hacer de la siguiente forma

```
kubectl delete deployment,service,pvc --all
kubectl delete secret mysql-pass
gcloud container clusters delete wordpress-cluster
```
Es recomendable limpiar todo al terminar, ya que si sigue levantado seguira facturando.

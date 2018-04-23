# POC Deploy Wordpress con Autoscaling y HA en Kubernetes

#### Introduccion

Prueba de concepto sobre una arquitectura de microservicios corriendo en un cluster de Kubernetes, contemplando escalabilidad y alta disponibilidad. 
Para esto se usara Wordpress como frontend y backend con dos nodos inicialmente, y una base de datos MySQL Master con dos Slave.

#### Prerequisitos

Crear un ambientes kubernetes corriendo con Minikube.

#### Ambiente

- Minikube v0.26.1
- Kubernetes v1.10
- Ubuntu v16.04/amd64

#### Objetivos

1. Crear volumenes persistentes 
2. Crear un "secret" para proteger datos sensibles
3. Crear y deployar una base de datos Mysql Master con dos Slave
4. Crear y deployar Wordpress
5. Testear la replicacion de la base de datos
6. Testear la aplicacion
7. Testear escalabilidad horizontal Webserver
8. Testear escalabilidad horizontal Base De Datos
9. Configurar Autoscaling Webserver
10. Testear Alta Disponibilidad Webserver
11. Testear Alta Disponibilidad Base De Datos

------



##### 1. Crear volumenes persistentes 

Para guardar la informacion y que persista al ciclo de vida de un pod crearemos volumenes persistentes:

```
$ kubectl create -f volumes.yml
```



##### 2. Crear un "secret" para proteger datos sensibles

```
$ kubectl create secret generic mysql-pass --from-literal=password=XXXXXXXX
```



##### 3. Crear y deployar una base de datos MySQL Master con dos Slave

Primero creamos un ConfigMap, esto seran variables de entorno que compartiran los containers, en este caso lograremos que en el master se pueda leer y escribir, y que replique a los slave que seran de solo lectura.

```
$ kubectl apply -f mysql-configmap.yml
```

Luego creamos el servicio de mysql.

```
$ kubectl apply -f mysql-service.yml
```

Por ultimo creamos el StatefulSet, que sera el encargado de iniciar los pods y tener las configuraciones necesarios para que se produzca la replica en los slave.

```
$ kubectl apply -f mysql-statefulset.yml
```



##### 4. Crear y deployar Wordpress

```
$ kubectl apply -f wordpress.yml
```



##### 5. Testear la replicacion de la base de datos

Levantando un container temporal podemos escribir en el master (mysql-0.mysql)

```
kubectl run mysql-client --image=mysql:5.7 -i --rm --restart=Never --\
  mysql -h mysql-0.mysql <<EOF
CREATE DATABASE test;
CREATE TABLE test.messages (message VARCHAR(250));
INSERT INTO test.messages VALUES ('hello');
EOF
```

Usando el hostname mysql-read podemos ejecutar una query en cualquier server que este disponible

```
kubectl run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never --\
  mysql -h mysql-read -e "SELECT * FROM test.messages"
```

Si la replicacion esta OK, mostrara el mensaje "hello" en la tabla "messages"

```
+---------+
| message |
+---------+
| hello   |
+---------+
```

Para demostrar que el servicio "mysql-read" distribuye las conexiones a todos los servers, se puede ejecutar el siguiente loop:

```
kubectl run mysql-client-loop --image=mysql:5.7 -i -t --rm --restart=Never --\
  bash -ic "while sleep 1; do mysql -h mysql-read -e 'SELECT @@server_id,NOW()'; done"
```

Esto mostrara que el server_id va cambiando de forma random, ya que en cada conexion elije un endpoint diferente



##### 6. Testear la aplicacion

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



##### 7. Testear escalabilidad horizontal Webserver

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



##### 8. Testear escalabilidad horizontal Base De Datos

Se puede aumentar la cantidad slaves logrande aumentar la capacidad de lectura de query

```
kubectl scale statefulset mysql  --replicas=4
```

Vericamos la creacion del nuevo pod,

```
kubectl get pods -l app=mysql -w
```

Tambien se puede verificar si el nuevo pod tiene la misma data que los otros

```
kubectl run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never --\
  mysql -h mysql-3.mysql -e "SELECT * FROM test.messages"
```



##### 9. Configurar autoscaling Webserver 

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



##### 10. Testear Alta Disponibilidad Webserver

Kubernetes por default brinda alta disponibilidad, esto lo logra implementando componentes de auto regeneracion. Esto se puede comprobar eliminando uno de los pods.

```
$ kubectl delete pod mysql
```



##### 11. Testear Alta Disponibilidad Base De Datos

Si un pod es eliminado, se recrea automaticamente uno nuevo con el mismo nombre, y linkeado a su PVC, tambien mantiene el id

```
kubectl delete pod mysql-2
```


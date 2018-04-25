# Configurar Mysql Master con Slaves

Para poder configurar instancias mysql que tengan alta disponibilidad hay que usar los Statefulsets que brinda Kubernetes, basicamente son similares a los Deployments, solo que ofrecen mejoras para aplicaciones como bases de datos en HA que necesitan iniciar en un cierto orden (al inicializarse primero master luego los slave) y un unico storage por cada pod (esta opcion esta en Beta actualmente)

Esta prueba sera levantar un nodo de MySQL master y dos slaves.

#### Objetivos

1. Crear y deployar una base de datos MySQL Master con dos Slave
2. Testear la replicacion 
3. Testear escalabilidad horizontal 

------



##### 1. Crear y deployar una base de datos MySQL Master con dos Slave

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



##### 2. Testear la replicacion 

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

Esto mostrara que el server_id va cambiando de forma random, ya que en cada conexion elije un endpoint diferente.



##### 3. Testear escalabilidad horizontal 

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
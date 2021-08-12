# kubernetes_volumenes_NFS
Como montar una partición NFS dentro de un pod.


En este artículo hablé de forma muy resumida sobre los volúmenes persitentes en Docker. En esta ocasión vamos a ver cómo gestionar dichos volúmenes en Kubernetes.
Los contenedores en Docker son transitorios por naturaleza y son destruidos una vez completan su función. El mismo principio se aplica a los datos del contenedor, los cuales son eliminados junto con el contenedor.  
El concepto por tanto de Persistent Volume es un volumen que contiene datos procesados por los contenedores y no son eliminados cuando los contenedores son destruidos. Por lo tanto, ¿cómo funcionan los PV en el mundo Kubernetes?.

![k8s_pv_pvc_storage](https://user-images.githubusercontent.com/18565089/129227386-38373b9d-faca-4206-812b-3804e6bb23f4.png)


Como en el mundo Docker, los Pods en Kubernetes son de naturaleza temporal y por lo tanto los datos son procesados y eliminados cuando el Pod se destruye.
Para evitar que los datos sean eliminados, se añade o “monta” un volumen al Pod cuando éste se crea, haciendo posible que los datos se mantengan después de que el Pod sea eliminado.
Aquí una muy simple implementación de un volumen montado como un directorio local, en un entorno muy simple con un único nodo. La configuración del Pod podría ser la siguiente:

```
apiVersion: v1
kind: Pod
metadata: 
  name: example
spec: 
  containers: 
  -  image: alpine
     name: alpine
     command: [“/bin/sh”,”-c”]
     args: [“shuf -I 0-100 -n 1 >> /opt/number.out;”]
     volumeMounts:
     -	mountPath: /opt
        name: data
  volumes:
  -  name: name_volume
     hostPath:
     path: /data
     type: Directory
```
De ésta forma, los ficheros procesados y almacenados en el directorio /opt dentro del Pod son almacenados en el directorio /data del host, impidiendo que sean eliminados cuando el Pod sea destruido.

Obviamente ésta es una configuration muy simple con un único host, y no es recomendable para escenarios mulit-hosts, dado que el Pod utilizará la ruta /data de todos los hosts esperando encontrar los mismos ficheros en todos los hosts y tendríamos que recurrir a algún tipo de replicación externa. 

Kubernetes soporta muchos tipos de storage solutions externos como por ejemplo: NFS, GlusterFS, Flocker, Ceph, Scaleio, y soluciones cloud como AWS, GCP, Azure y OCI.

Ejemplo con AWS EBS:

```
volumes:
-  name: data
   awsElasticBlockStore:
     volumeID: <volumen-id>
     fsType: ext4
```

¿Qué ocurre en un entorno más complejo con varios hosts, numerosos usuarios, pods, etc.?
Lo ideal es poder gestionar la configuración de almacenamiento de forma centralizada y que los administradores puedan crear un gran storage pool y los usuarios o ingenieros DevOps puedan tomar porciones de dicho pool para sus aplicaciones.

En éste punto es cuando entran en escena los conceptos PV (Persistent Volume) y PVC (Persistent Volume Claim).

Los administradores crean los Persisten Volumes según tamaño, tier, etc.. y los usuarios los PVC o claims según las necesidades de las aplicaciones, ajustándose a los PV existentes.

Definicion-PV.yaml:

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 name: pv-vol1
spec:
 accessModes:
 - ReadWriteOnce
 capacity:
   storage: 1Gi
 hostPath:
    path: /tmp/data
```

```
(recordatorio, el almacenamiento local no es adecuada para entornos de producción)
```

Los valores admitidos para el accessmode son: ReadOnlyMany, ReadWriteOnce y ReadWriteMany.

Ejemplos utilizando otras soluciones de Storage:

AWS EBS:

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 name: pv-vol1
spec:
 accessModes:
 - ReadWriteOnce
 capacity:
   storage: 1Gi
 awsElasticBlockStorage:
   volumeID: <volume-id>
   fsType: ext4
```

**Persistent Volumes (PV) y Persistent Volumes Claims (PVC)** son objetos diferentes en K8s. Una vez la PVC es creado, K8s asignará el PV que se ajuste a los requerimientos y otros factores al PVC.

Cada PVC es asignado a un único PV.

K8s asigna un PV a un PVC en función de los siguientes factores:

- Capacidad suficiente.
- Modo de acceso.
- Modos del volumen
- Storage Class.
- Labels y Selectors (con el fin de asignar el PV que se quiera).
- 
En el caso de que no haya una opción mejor, K8s puede asignar PV de mayor capacidad al PVC, siendo en éste caso poco eficiente.

En el caso de que K8s no encuentre un PV que pueda asignar al PVC, el PVC quedará en estado “Pending”, esperando que haya un nuevo PV disponible.

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 name: pvc-example
spec:
 accessModes:
 - ReadWriteMany
 resources:
  requests:
    storage: 500Mi

```

¿Que ocurre con el PV cuando se elimina el PVC?. Se puede elegir el comportamiento con las siguientes opciones:

```
persistentVolumeReclaimPolicy: Retain (El PV no será eliminado y requiere de la intervención del administrador y no estará disponible para su utilización por cualquier otro PVC).
persistentVolumeReclaimPolicy: Delete (el PV será eliminado automáticamente). persistentVolumeReclaimPolicy: Recycle (Los datos serán eliminados pero el PV puede ser utilizado por otro PVC). 

```

Los ejemplos que hemos visto son para configuraciones Static Provisioning, que requieren de la creación de PV cada vez. Para una asignación dinámica se utiliza Dyamic Provisioning, haciendo uso de Storage Classes, de los que hablaré en otro post.

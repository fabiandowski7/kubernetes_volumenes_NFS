# Kubernetes: Volúmenes NFS

Soy Oscar Mas y cuando eliminamos un pod (contenedor), toda la información que hay en él desaparece. Para poder conseguir que los datos que se han ido almacenando en nuestro pod persistan (no se eliminen al borrar el pod), es necesario usar los volúmenes de docker (Data Volume)

Hoy os quiero enseñar como montar una particion NFS dentro de un pod.

Para ello montaremos un servidor NFS (ub-nfs-sbd) y posteriormente desplegaremos los volúmenes, para poder acceder al servicio de NFS desde nuestros pods

El esquema lógico sería el siguiente:

![kubernetes-015](https://user-images.githubusercontent.com/18565089/129230886-02bd57a5-c655-4c43-a053-65464f031b38.png)

El primer paso que haremos es instalar el sistema de NFS en el servidor ub-nfs-sbd y posteriormente compartiremos la carpeta en el servidor ub-nfs-sbd para que los dos pods puedan montar dicha carpeta.

```
root@ub-nfs-sbd:~# apt-get install -y nfs-common nfs-kernel-server
root@ub-nfs-sbd:~# cat /etc/exports
/home/nfs               *(rw,no_root_squash)
root@ub-nfs-sbd:~# mkdir /home/nfs
root@ub-nfs-sbd:~# echo "NFS Kubernetes" > /home/nfs/oscarmas
root@ub-nfs-sbd:~# systemctl restart nfs-kernel-server
```

Una vez montado nuestro servidor de NFS, es necesario que instalemos el cliente de NFS en los servidores que realizan las funciones de cliente de Kubernetes (ub-nodo1-sbd y ub-nodo2-sbd). Esto lo realizaremos mediante el siguiente comando:

```
root@ub-nodo1-sbd:~# apt-get install -y nfs-common
```
![kubernetes-016-694x472](https://user-images.githubusercontent.com/18565089/129231126-c215d5cc-e072-4d66-8b79-2340378a0590.png)

Verificaremos la conectividad con nuestro servidor de NFS desde cualquiera de los dos servidores (ub-nodo1-sbd y ub-nodo2-sbd)

![kubernetes-017-694x226](https://user-images.githubusercontent.com/18565089/129231191-3e27a8f0-0b7e-49b1-8dd8-ea3b94ce35e5.png)

Una vez verificado, crearemos un yaml de pv y pvc, desde nuestro servidor de Kubernetes y verificaremos su correcto funcionamiento:

```
root@ub-nfs-sbd:~# apt-get install -y nfs-common nfs-kernel-server
root@ub-nfs-sbd:~# cat /etc/exports
/home/nfs               *(rw,no_root_squash)
root@ub-nfs-sbd:~# mkdir /home/nfs
root@ub-nfs-sbd:~# echo "NFS Kubernetes" > /home/nfs/oscarmas
root@ub-nfs-sbd:~# systemctl restart nfs-kernel-server
```
Crearemos el PV y el PVC de la siguiente manera, posteriormente verificaremos que se han creado:

```
rootdevel@ub-nodo0-sbd:~$ kubectl create -f nfs-pv-pvc.yaml
rootdevel@ub-nodo0-sbd:~$ kubectl get pv
rootdevel@ub-nodo0-sbd:~$ kubectl get pvc
```

Una vez hecho, solamente nos queda crear un par de pods enlazados con nuestros PV y PVC

```
rootdevel@ub-nodo0-sbd:~$ cat nfs-nginx-rc.yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: nfs-web
spec:
  replicas: 2
  selector:
    role: nfs-web-rc
  template:
    metadata:
      labels:
        role: nfs-web-rc
    spec:
      containers:
      - name: web
        image: nginx
        ports:
          - name: web
            containerPort: 80
        volumeMounts:
            - name: kube-nfs-pvc
              mountPath: "/usr/share/nginx/html"
      volumes:
      - name: kube-nfs-pvc
        persistentVolumeClaim:
          claimName: kube-nfs-pvc
```

Crearemos los pods y posteriormente verificaremos que se ha montado correctamente la partición NFS en uno de ellos:

```
rootdevel@ub-nodo0-sbd:~$ kubectl create -f nfs-nginx-rc.yaml
rootdevel@ub-nodo0-sbd:~$ kubectl get rc -o wide
```
Seguidamente nos conectaremos a uno de los pods, para verificar que se ha montado la partición correctamente:

```
rootdevel@ub-nodo0-sbd:~$ kubectl get pods -o wide
rootdevel@ub-nodo0-sbd:~$ kubectl exec -it nfs-web-fscpl -- /bin/bash
root@nfs-web-fscpl:/# df -h
root@nfs-web-fscpl:/# cat /usr/share/nginx/html/oscarmas
root@nfs-web-fscpl:/# exit
```
![kubernetes-018-694x417](https://user-images.githubusercontent.com/18565089/129231507-8c1c9218-23db-4f6e-81c5-558e7700fee9.png)


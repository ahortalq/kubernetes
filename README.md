# kubernetes

## Creación de un cluster K8s
En el directorio `vagrant-provisioning` está lo necesario para provisionar un **master** y dos **workers**. Basta iniciar las instancias con ...
```
cd vagrant-provisioning
vagrant up
```
Esto creará tres máquinas viruales con un cluster K8s montado.

### Configurar el acceso vía kubectl
#### Crear nuevas entradas en /etc/hosts
```
172.42.42.100 kmaster.example.com kmaster
172.42.42.101 kworker1.example.com kworker1
172.42.42.102 kworker2.example.com kworker2
```
#### Copiamos la configuración del master bajo ~/.kube
La password del usuario root es *kubeadmin*.
```
$ mkdir ~/.kube
$ scp root@kmaster.example.com:/etc/kubernetes/admin.conf ~/.kube
$ mv ~/.kube/admin.conf ~/.kube/config
$ kubectl cluster-info
Kubernetes master is running at https://172.42.42.100:6443
KubeDNS is running at https://172.42.42.100:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```
#### Verificación de los nodos
```
$ kubectl get nodes
NAME                   STATUS   ROLES    AGE   VERSION
kmaster.example.com    Ready    master   38m   v1.15.1
kworker1.example.com   Ready    <none>   36m   v1.15.1
kworker2.example.com   Ready    <none>   34m   v1.15.1
```
#### Instalación del Dashboard Web UI en K8s
Primero veamos los pods que hay en ejecución en el namespace `kube-system`
```
$ kubectl -n kube-system get pods -o wide
NAME                                          READY   STATUS    RESTARTS   AGE   IP              NODE                   NOMINATED NODE   READINESS GATES
coredns-5c98db65d4-9jw4f                      1/1     Running   0          46m   10.244.0.3      kmaster.example.com    <none>           <none>
coredns-5c98db65d4-cnknq                      1/1     Running   0          46m   10.244.0.2      kmaster.example.com    <none>           <none>
etcd-kmaster.example.com                      1/1     Running   0          45m   172.42.42.100   kmaster.example.com    <none>           <none>
kube-apiserver-kmaster.example.com            1/1     Running   0          45m   172.42.42.100   kmaster.example.com    <none>           <none>
kube-controller-manager-kmaster.example.com   1/1     Running   0          45m   172.42.42.100   kmaster.example.com    <none>           <none>
kube-flannel-ds-amd64-j2fg2                   1/1     Running   0          43m   172.42.42.102   kworker2.example.com   <none>           <none>
kube-flannel-ds-amd64-ld7c5                   1/1     Running   0          46m   172.42.42.100   kmaster.example.com    <none>           <none>
kube-flannel-ds-amd64-z72j7                   1/1     Running   0          45m   172.42.42.101   kworker1.example.com   <none>           <none>
kube-proxy-2cd9k                              1/1     Running   0          46m   172.42.42.100   kmaster.example.com    <none>           <none>
kube-proxy-bk7n9                              1/1     Running   0          45m   172.42.42.101   kworker1.example.com   <none>           <none>
kube-proxy-jwq6q                              1/1     Running   0          43m   172.42.42.102   kworker2.example.com   <none>           <none>
kube-scheduler-kmaster.example.com            1/1     Running   0          45m   172.42.42.100   kmaster.example.com    <none>           <none>
```

Ahora instalar el Dashboard. Ir al directorio `dashboard` y ejecutar:
```
$ cd dashboard
$ kubectl create -f influxdb.yaml
deployment.extensions/monitoring-influxdb created
service/monitoring-influxdb created
```
Si volvemos a consultar los pods del namespace `kube-system` veremos que hay nuevos pods en ejecución. Repetimos lo mismo con `heapster.yaml`
```
$ kubectl create -f heapster.yaml
erviceaccount/heapster created
clusterrolebinding.rbac.authorization.k8s.io/heapster-rolebinding created
deployment.extensions/heapster created
service/heapster created
```
Y lo mismo con `dashboard.yaml`
```
$ kubectl create -f dashboard.yaml
secret/kubernetes-dashboard-certs created
serviceaccount/kubernetes-dashboard created
role.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
deployment.apps/kubernetes-dashboard created
service/kubernetes-dashboard created
```
Si ahora hacemos de nuevo `kubectl -n kube-system get pods -o wide` veremos en qué máquina se ha instalado el dashboard. Podemos acceder a él con la siguiente URL: `https://kworker1.example.com:32323`. Ahora necesitamos crear una `service account` para acceder vía **token**. Para ello ejecutamos ...
```
$ kubectl create -f sa_cluster_admin.yaml
serviceaccount/dashboard-admin created
clusterrolebinding.rbac.authorization.k8s.io/cluster-admin-rolebinding created
```
Vamos a obtener el token creado. Para ello ...
```
$ # Obtenemos el ID secret del service account
$ kubectl describe sa dashboard-admin -n kube-system
Name:                dashboard-admin
Namespace:           kube-system
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   dashboard-admin-token-xcbdj
Tokens:              dashboard-admin-token-xcbdj
Events:              <none>
$ # Obtenemos los detalles del secret
$ kubectl describe secret dashboard-admin-token-xcbdj -n kube-system
Name:         dashboard-admin-token-xcbdj
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: dashboard-admin
              kubernetes.io/service-account.uid: 72edfd43-949a-40c9-bcde-80c080ae8a85

Type:  kubernetes.io/service-account-token

Data
====
token:      eyJhbGci..._QXMvmjq3j7r-ovAQ
ca.crt:     1025 bytes
namespace:  11 bytes
```
Con el `token` obtenido, lo pegamos en la UI del Dashboard y ya tenemos acceso.
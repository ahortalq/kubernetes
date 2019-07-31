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

## Despliegue de un contenedor Docker en el cluster
Para desplegar un simple contenedor en el cluster, podemos hacer:
```
$ kubectl run myshell -it --image busybox -- sh
```
Para eliminaro:
```
$ kubectl delete deploy myshell
```

Arranquemos ahora un contenedor `nginx`
```
$ kubectl run nginx --image nginx
```
Ahora vamos a acceder al servicio nginx sin necesidad de crear un `service`. Esto lo hacemos con un `port-forward`.
```
$ kubectl port-forward nginx-7bb7cd8db5-ktpch 6789:80
```
Con esto podremos acceder a Nginx con http://localhost:6789. Para ver los **logs** del contenedor podemos ejecutar:
```
$ kubectl logs nginx-7bb7cd8db5-ktpch
```
Para aumentar el número de **réplicas**, es decir, el número de contenedores en ejecución, ejecutamos uno de los siguientes comandos:
```
$ kubectl run nginx --image nginx --replicas 2
$ kubectl scale deploy nginx --replicas 2
```
Vamos a crear un **servicio** (en lugar de hacer un port-forward). Para ello:
```
$ kubectl expose deployment nginx --type NodePort --port 80
```
Ahora vamos a generar un fichero yaml con la descripción de nginx, el puerto, el servicio, etc. Para ello:
```
$ kubectl get deploy nginx -o yaml > /tmp/nginx.yaml
$ kubectl get svc nginx -o yaml > /tmp/svc.yaml
```
Luego podemos aplicar esos recursos definidos en el yaml de la siguiente forma:
```
$ kubectl create -f /tmp/nginx.yaml
$ kubectl create -f /tmp/svc.yaml
```
Y para borrar recursos ...
```
$ kubectl delete -f /tmp/nginx.yaml
$ kubectl delete -f /tmp/svc.yaml
```
### Para monitorizar el estado
```
$ watch kubectl get all -o wide
```
## Creación de Pods, ReplicaSets and Deployments
### Creación de Pod
Un pod en K8s es la unidad más pequeña de ejecución. Por ejemplo, creamos el fichero `01-nginx-pod.yaml`, este será el contenido mínimo:
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
```
Se ejecutaría con `kubectl create -f 01-nginx-pod.yaml`. Y para elminarlo con `kubectl delete -f 01-nginx-pod.yaml`.
### Creación de ReplicaSet
Con esto indicaremos cuántos pods queremos en el cluster (también llamado `Replication Controller`). Cuidado con el `matchLables`, este ReplicaSet gestionará todos los pods que tengan el label `run` con el valor `nginx`. Creamos el fichero `01-nginx-replicaset.yaml`.
```
apiVersion: extensions/v1beta1
kind: ReplicaSet
metadata:
  name: nginx-replicaset # Nombre del ReplicaSet
spec:
  replicas: 2
  selector:
    matchLabels:
      run: nginx # Este ReplicaSet manejará todos los pods que
                 # cumplan este criterio
  template:
    metadata:
      labels:
        run: nginx # Esto debe coincidir con el matchLabels
                   # definido en el selector
    spec:
      containers:
      - image: nginx
        name: nginx
```
### Creación de un Deployment
La ventaja de usar Deployments sobre ReplicaSets es que cuando se despliega la aplicación, podemos elegir entre varias estrategias de despliegue. Cuando se crea un `Deployment`, automáticamente se crea también un `ReplicaSet`. Creamos el fichero `01-nginx-deployment.yaml` con el siguiente contenido. En principio es muy parecido al código del `ReplicaSet`.
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  replicas: 2
  selector:
    matchLabels:
      run: nginx
  template:
    metadata:
      labels:
        run: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
```
## Namespaces y Contexts
### Obtener una lista de los `namespaces` disponibles:
```
$ kubectl get namespaces
NAME              STATUS   AGE
default           Active   19h
kube-node-lease   Active   19h
kube-public       Active   19h
kube-system       Active   19h
```
### Creación de un ns
```
$ kubectl create ns tutorial
namespace/tutorial created
```
### Creación de un Context
Para obtener la configuración actual ejecutamos:
```
$ kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://172.42.42.100:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
```
Ahí podremos ver los **contextos** disponibles y el que está en ejecución. Pero para obtener los contextos disponibles es más fácil ejecutar:
```
$ kubectl config get-contexts
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
*         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin
```
Para crear un nuevo `context` de nombre `tutorial-context`.
```
$ kubectl config set-context tutorial-context --namespace=tutorial --user=kubernetes-admin --cluster=kubernetes
Context "tutorial-context" created.
$ kubectl config get-contexts
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
*         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin   
          tutorial-context              kubernetes   kubernetes-admin   tutorial
```
Cambiemos el contexto.
```
$ kubectl config use-context tutorial-context
Switched to context "tutorial-context".
```
A partir de ahora, todos los recursos que creemos, modifiquemos, borremos, ... se aplicarán al namespace `tutorial` (que es el namespace asociado al contexto).

## Node Selector
Esto servirá para que ciertos pods se ejecuten en **ciertos nodos** del cluster. Puede ser que queramos ejecutar un pod en cierto nodo porque tiene más memoria, más CPU, etc.

Para ello tendríamos que asociar `labels` a los nodos. Para ello:
```
$ kubectl label node kworker2.example.com demoserver=true
node/kworker2.example.com labeled
```
Para verificar que pusimos bien el `label`tendríamos que ejecutar:
```
$ kubectl get node kworker2.example.com --show-labels
NAME                   STATUS   ROLES    AGE   VERSION   LABELS
kworker2.example.com   Ready    <none>   22h   v1.15.1   ...,demoserver=true,...
```
Ahora veamos cómo modificar el `deployment` para hacer uso de ese `nodo` y ese `label`.
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  replicas: 2
  selector:
    matchLabels:
      run: nginx
  template:
    metadata:
      labels:
        run: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
      nodeSelector:             # Esta sería la nueva línea a incluir
        demoserver: "true"
```
## DaemonSets
Un DeamonSet es un pod que se despliega en todos los nodos del cluster. **Un pod por nodo**. Si se agrega un nuevo nodo al cluster, se desplegará el pod en ese nodo. Si un nodo muere, también desaparece ese pod (no se replanifica para ejecutarse en otro nodo).

Aunque pueden desplegarse estos `DaemonSets` en determinados nodos, por ejemplo, nodos con cierto `label`.

Creamos el fichero `01-nginx-daemonset.yaml` con el siguiente contenido:
```
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: nginx-daemonset
spec:
  selector:
    matchLabels:
      demotype: nginx-daemonset-demo
  template:
    metadata:
      labels:
        demotype: nginx-daemonset-demo
    spec:
      containers:
      - image: nginx
        name: nginx
```
Si queremos determinar en qué nodos debe ejecutarse este DaemonSet tendremos que incluir nuevas líneas
```
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: nginx-daemonset
spec:
  selector:
    matchLabels:
      demotype: nginx-daemonset-demo
  template:
    metadata:
      labels:
        demotype: nginx-daemonset-demo
    spec:
      containers:
      - image: nginx
        name: nginx
      nodeSelector:     # Aquí indicamos dónde ejecutarse el DaemonSet
        demoserver: "true"
```
## Jobs & CronJobs
### Jobs
Un job es algo que se ejecuta en el cluster. Por ejemplo, creamos el fichero `02-jobs.yaml` con el siguiente contenido.
```
apiVersion: batch/v1
kind: Job
metadata:
  name: helloworld
spec:
  completions: 2    # Número de veces que debe terminar el job
                    # Se ejecuta de forma secuencial
  template:
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ["sleep", "60"]
      restartPolicy: Never
```
Si queremos ejecutar los jobs en paralelo tendríamos que incluir una nueva línea indicando el máximo número de jobs que podemos ejecutar en paralelo.
```
apiVersion: batch/v1
kind: Job
metadata:
  name: helloworld
spec:
  completions: 6
  parallelism: 2  # Se ejecutarán como máximo dos jobs en paralelo.
  template:
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ["sleep", "60"]
      restartPolicy: Never
```
Es posible que la ejecución del job finalice con error, el job intentará crear un nuevo pod hasta que se alcance el nº de `completions` configurado en el job. Para eso tenemos el parámetro `backoffLimit`. Si se crean tantos pods como se indica en este parámetro y el job no finalizó con éxito, se cancela el job.
```
apiVersion: batch/v1
kind: Job
metadata:
  name: helloworld
spec:
  backoffLimit: 4   # Indicamos el máximo nº de intentos
  template:
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ["ls", "/jcla"]    # Esto fallará
      restartPolicy: Never
```
Si queremos dar un tiempo límite para la ejecución del job, utilizaremos `activeDeadlineSeconds`.
```
apiVersion: batch/v1
kind: Job
metadata:
  name: helloworld
spec:
  activeDeadlineSeconds: 10 # Tiempo límite para ejecutar el job
  template:
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ["sleep", "60"]   # Tardará más y fallará
      restartPolicy: Never
```
### CronJobs
Son iguales que los jobs pero que se ejecutan de forma periódica. Como ejemplo tenemos el fichero `02-cronjob.yaml`:
```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: helloworld-cron
spec:
  schedule: "* * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: busybox
            image: busybox
            command: ["echo", "Hello Kubernetes!!!"]
          restartPolicy: Never
```
En este caso, cada minuto se creará un nuevo job que será el encargado de hacer el trabajo. Hay algunas propiedades y parámetros adicionales que no vamos a ver.
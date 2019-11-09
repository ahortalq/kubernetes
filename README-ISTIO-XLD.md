# Configuración XLD, Istio y K8s

Vamos a ver lo necesario para configurar XLD, Istio y K8s.

## 1 Creación de deployment en XLD con un solo contenedor y un solo servicio

Esto es lo mínimo que se necesita para un deployment. Es fácil llevarlo a XLD.
```
apiVersion: apps/v1
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

Esto es lo mínimo que se necesita para un service.
```
apiVersion: v1
kind: Service
metadata:
  name: vote-service
spec:
  type: ClusterIP
  selector:
    run: vote
  ports:
    - port: 80
```

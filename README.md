# Develop Wordpress with Google Kubernetes Engine (TLS/SSL termination)

This is a guide to help you to develop your Wordpress in Google Kubernetes Engine and with

  - TLS/SSL termination
  - Loading balance
  - Using Persistent Disks

### 1. Setup your kubernetes cluster and connect

Setup your kubernetes cluster at Google Kubernetes Engine.

After that, connect kubernetes cluster with your kubectl.
```sh
$ gcloud container clusters get-credentials yourk8s --zone asia-east1-b --project yourk8s
```
You can check your connecting with nodes.
```sh
$ kubectl get nodes
```

### 2. Create Persistent Volumes for MySQL

`mysql-volumeclaim.yaml`
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: mysql-volumeclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
```
Create Persistent Volumes for MySQL with `mysql-volumeclaim.yaml`.
```sh
$ kubectl create -f mysql-volumeclaim.yaml
```

### 3. Create Persistent Volumes for Wordpress

`wordpress-volumeclaim.yaml`
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: wordpress-volumeclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
```
Create Persistent Volumes for Wordpress with `wordpress-volumeclaim.yaml`.
```sh
$ kubectl apply -f wordpress-volumeclaim.yaml
```

### 4. Create Secret for MySQL password

```sh
$ kubectl create secret generic mysql --from-literal=password=root
```

### 5. Create MySQL Deployment

`mysql.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - image: mysql:5.6
          name: mysql
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql
                  key: password
          ports:
            - containerPort: 3306
              name: mysql
          volumeMounts:
            - name: mysql-persistent-storage
              mountPath: /var/lib/mysql
      volumes:
        - name: mysql-persistent-storage
          persistentVolumeClaim:
            claimName: mysql-volumeclaim
```
Create MySQL Deployment with `mysql.yaml`.
```sh
$ kubectl create -f mysql.yaml
```

### 6. Create MySQL Service

`mysql-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  type: ClusterIP
  ports:
    - port: 3306
  selector:
    app: mysql
```
Create MySQL Service with `mysql-service.yaml`.
```sh
$ kubectl create -f mysql-service.yaml
```

### 7. Create Wordpress Deployment

`wordpress.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
        - image: wordpress
          name: wordpress
          env:
          - name: WORDPRESS_DB_HOST
            value: mysql:3306
          - name: WORDPRESS_DB_PASSWORD
            valueFrom:
              secretKeyRef:
                name: mysql
                key: password
          ports:
            - containerPort: 80
              name: wordpress
          volumeMounts:
            - name: wordpress-persistent-storage
              mountPath: /var/www/html
      volumes:
        - name: wordpress-persistent-storage
          persistentVolumeClaim:
            claimName: wordpress-volumeclaim
```
Create Wordpress Deployment with `wordpress.yaml`.
```sh
$ kubectl create -f wordpress.yaml
```

### 8. Create Wordpress Service

`wordpress-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: wordpress
  name: wordpress
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  selector:
    app: wordpress
```
Create Wordpress Service with `wordpress-service.yaml`.
```sh
$ kubectl create -f wordpress-service.yaml
```

### 9. Increase the Maximum File Upload Size (Option)

Get your wordpress pod name and copy.
```sh
$ kubectl get pod -l app=wordpress
```
Go into the wordpress pod container.
```sh
$ kubectl exec -ti wordpress-78c9b8d684-zvnqf bash
```
Install `vim` package.
```sh
$ apt-get update
$ apt-get install vim
```
Modify `.htaccess` file. Put the code at the bottom of the file.
```
php_value upload_max_filesize 128M
php_value post_max_size 128M
php_value memory_limit 256M
php_value max_execution_time 300
php_value max_input_time 300
```

### 10. Create Secret for TLS key

Make sure your Wordpress is workable and finish installation.

And then, you can start input TLS key.
```sh
$ kubectl create secret tls yourwp-tls --key tls/wp.key --cert tls/wp.crt
```
`ingress.yaml`
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress
spec:
  tls:
  - hosts:
    - yourwp.com
    secretName: yourwp-tls
  rules:
    - host: yourwp.com
      http:
        paths:
        - path: /
          backend:
            serviceName: wordpress
            servicePort: 80
        - path: /*
          backend:
            serviceName: wordpress
            servicePort: 80
```
Create Ingress Controller with `ingress.yaml`.
```sh
$ kubectl create -f ingress.yaml
```
Now, you can get the static IP with Ingress Controller at [GCP networking](https://console.cloud.google.com/networking/addresses/).

Copy your static IP and set the DNS.

After DNS is working, login the Wordpress admin and change the default URL and start with [Really Simple SSL](https://tw.wordpress.org/plugins/really-simple-ssl/)

### Reference

[Kubernetes Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

[Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)

[Using Persistent Disks with WordPress and MySQL](https://cloud.google.com/kubernetes-engine/docs/tutorials/persistent-disk#deploy_mysql)

[Setting up HTTP Load Balancing with Ingress](https://cloud.google.com/kubernetes-engine/docs/tutorials/http-balancer)

[Really Simple SSL](https://tw.wordpress.org/plugins/really-simple-ssl/)

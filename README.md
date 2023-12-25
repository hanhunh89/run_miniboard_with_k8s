# run_miniboard_with_k8s

## introduction

1. Make miniboard project with spring.<br>
https://github.com/hanhunh89/spring-miniBoard-cluster<br><br>

2. Install and study kubernetes<br>
https://github.com/hanhunh89/k8s_install <br>
https://github.com/hanhunh89/k8s_study <br>

3. Now, run miniboard in kubernetes(On-Premises)  ! 


# make db pod. 

## secret.yaml
encorde root password to base64.<br>
```
$ echo -n 1234 | base64
MTIzNA==
```
create secret.yaml for saving root password
```
# mariadb_root_pw_secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mariadb-root-password
data:
  password: MTIzNA==  # secreat value needs base64 encording
```
```
kubectl apply -f mariadb_root_pw_secret.yaml 
```
## create configmap for initialize db.
```
# miniboard-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mariadb-initdb-config
data:
  init.sql: |
    CREATE DATABASE IF NOT EXISTS myDB;   
    USE myDB;		
    CREATE TABLE post (postId INT AUTO_INCREMENT PRIMARY KEY,title VARCHAR(30),content TEXT,view INT DEFAULT 0,writer VARCHAR(30),imageName VARCHAR(50));
    CREATE TABLE user (userId VARCHAR(50) NOT NULL PRIMARY KEY,password VARCHAR(100) NOT NULL,created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,auth VARCHAR(20) DEFAULT 'ROLE_USER');
    CREATE USER 'myuser'@'%' IDENTIFIED BY '1234'; 
    GRANT ALL PRIVILEGES ON myDB.* TO 'myuser'@'%' WITH GRANT OPTION; 	
    FLUSH PRIVILEGES;
```
```
kubectl apply -f miniboard-configmap.yaml
```

## create service
```
# miniboard-mariadb-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: mariadb-deployment
spec:
  ports:
  - port: 3306
  selector:
    app: mariadb
```
```
kubectl apply -f  miniboard-mariadb-service.yaml
```

## create volume for db data
```
sudo mkdir /dbdata
```
```
#pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: db-pv
  labels:
    type: local
spec:
  storageClassName: my-storage
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: "/dbdata" 
```
```
kubectl apply -f pv.yaml
```
```
#pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-pv-claim
spec:
  storageClassName: my-storage
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```
```
kubectl apply -f pvc.yaml
```

## create mariadb deployment
```
# mariadb-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mariadb-deploy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mariadb
  template:
    metadata:
      labels:
        app: mariadb
    spec:
      containers:
      - name: mariadb
        image: mariadb:latest
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mariadb-root-password
              key: password
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mariadb-init-sql
          mountPath: /docker-entrypoint-initdb.d 
        - name: db-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mariadb-init-sql
        configMap:
          name: mariadb-initdb-config
      - name: db-storage
        persistentVolumeClaim:
          claimName: db-pv-claim
```
```
kubectl apply -f mariadb-deploy.yaml
```

## connect mariaDB
```
$ kubectl get pods
NAME                                 READY   STATUS    RESTARTS   AGE
mariadb-deployment-b5c6669d8-75knv   1/1     Running   0          2d3h

$ kubectl exec -it mariadb-deployment-b5c6669d8-75knv -- /bin/bash

$ mariadb -u root -p 
Enter password:  #insert root password(1234)

MariaDB [(none)]> use myDB;
MariaDB [myDB]> show tables;
+----------------+
| Tables_in_myDB |
+----------------+
| post           |
| user           |
+----------------+
```

# make was(tomcat) 
## create tomcat service
```
#tomcat-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: tomcat-service
spec:
  selector:
    app: tomcat
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
```
```
kubectl apply -f tomcat-svc.yaml
```

## create tomcat deployment
```
# tomcat-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tomcat-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: tomcat
  template:
    metadata:
      labels:
        app: tomcat
    spec:
      initContainers:
       - name: git-clone-init-container
         image: alpine/git:latest
         command: ['git', 'clone', 'https://github.com/hanhunh89/run_miniboard_with_k8s.git', '/git-repo']
         volumeMounts:
         - name: git-repo-volume
           mountPath: /git-repo
       - name: get-key-init-container
         image: busybox
         command: ['sh', '-c', 'wget -O /git-repo/key.json https://gcp_cloud_storage_key_file']
         volumeMounts:
         - name: git-repo-volume
           mountPath: /git-repo
      containers:
      - name: tomcat-container
        image: tomcat:9
        volumeMounts:
        - name: git-repo-volume
          mountPath: /usr/local/tomcat/webapps
      volumes:
      - name: git-repo-volume
        emptyDir: {}
```
```
curl -L <externalIP>:30000/miniboard
```
end. <br>

next, we will do this project in GCP and AWS. 

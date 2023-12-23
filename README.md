# run_miniboard_with_k8s

## introduction

1. Make miniboard project with spring.<br>
https://github.com/hanhunh89/spring-miniBoard<br><br>

2. Install and study kubernetes<br>
https://github.com/hanhunh89/k8s_install <br>
https://github.com/hanhunh89/k8s_study <br>

3. Now, run miniboard in kubernetes ! 


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
    CREATE USER 'myuser'@'%' IDENTIFIED BY '1234';  				#myuser 계정 생성 11.22.33.44 는 tomcat1 ip로 변경
    GRANT ALL PRIVILEGES ON myDB.* TO 'myuser'@'%' WITH GRANT OPTION; 	#11.22.33.44는 tomcat1 ip로 변경
    CREATE USER 'myuser'@'%' IDENTIFIED BY '1234';  				#myuser 계정 생성 55.55.55.55 는 tomcat2 ip로 변경
    GRANT ALL PRIVILEGES ON myDB.* TO 'myuser'@'55.55.55.55' WITH GRANT OPTION; 	#55.55.55.55는 tomcat2 ip로 변경
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
  name: mariadb
spec:
  ports:
  - port: 3306
  selector:
    app: mariadb
```
```
kubectl apply -f  miniboard-mariadb-service.yaml
```

## create mariadb deployment
```
# mariadb-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mariadb-deployment
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
      volumes:
      - name: mariadb-init-sql
        configMap:
          name: mariadb-initdb-config
```
```
kubectl apply -f mariadb-deploy.yaml
```

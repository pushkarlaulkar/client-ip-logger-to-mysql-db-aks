Instructions to deploy **Client IP Logger in MySQL DB** on Azure Kubernetes Service using the default **App Routing** add on
  1. Deploy AKS cluster through Azure portal.
  2. Deploy **MySQL Replication Cluster** ( 1 primary, 2 secondary ) helm chart. MySQL will be deployed as a **StatefulSet**. Set the **auth.rootPassword** & **auth.replicationPassword** of your choice.
     
     ```
     helm repo add bitnami https://charts.bitnami.com/bitnami
     helm repo update
     
     helm install mysql-cluster bitnami/mysql \
        --namespace db \
        --create-namespace \
        --set architecture=replication \
        --set auth.rootPassword= \
        --set auth.replicationPassword= \
        --set primary.persistence.enabled=true \
        --set primary.persistence.size=10Gi \
        --set primary.persistence.storageClass=managed-csi \
        --set secondary.replicaCount=2 \
        --set secondary.persistence.enabled=true \
        --set secondary.persistence.size=10Gi \
        --set secondary.persistence.storageClass=managed-csi \
        --set volumePermissions.enabled=true
     ```
  3. Create the Table in MySQL which will store the Client IP

     ```
     kubectl exec -it -n db mysql-cluster-primary-0 -- bash
     mysql -uroot -p ( Enter root user password when prompted )

     CREATE DATABASE IF NOT EXISTS flaskdb;
     USE flaskdb;

     CREATE TABLE IF NOT EXISTS client_ips ( id INT AUTO_INCREMENT PRIMARY KEY, ip_address VARCHAR(45) );
     ```
  4. Create a namespace **web** where the application will be deployed.
  5. Put the base64 encoded value of the **auth.rootPassword** set above in **mysql-secret.yaml** & then create the **Secret**.
     
     ```
     kubectl -n web apply -f mysql-secret.yaml
     ```
  6. Create the **ConfigMap** defined in **mysql-configmap.yaml**. The ConfigMap has the values for database host, name & user.
     
     ```
     kubectl -n web apply -f mysql-configmap.yaml
     ```
  7. Create the **Deployment** & **Service**. 

     ```
     kubectl -n web apply -f flask-deployment.yaml -f flask-service.yaml
     ```
  8. Create a tls secret named ` cert-tls ` which has the domain's certificate & private key by running below command. The domain's .crt & .key file should already be present.

     ```
     kubectl -n web create secret tls cert-tls --cert=domain_name.crt --key=domain_name.key
     ```
  9. Run the command ` kubectl -n app-routing-system get svc nginx ` to confirm if a **LoadBalancer** IP has been provisioned.

     ```
     pushkar [ ~ ]$ kubectl -n app-routing-system get svc nginx
     NAME    TYPE           CLUSTER-IP    EXTERNAL-IP    PORT(S)                      AGE
     nginx   LoadBalancer   10.0.58.180   20.174.45.64   80:32767/TCP,443:30598/TCP   110s
     ```
  8. Put the FQDN for which the secret has been created in ` app-routing-ingress.yml ` file and then run the command ` kubectl -n web apply -f app-routing-ingress.yml `.
  9. Run `kubectl -n web get ingress` to retrieve the IP. This may take some time to match with the **LoadBalancer** IP above. Point the domain name in your registrar to the IP address.
 10. Access the app using ` curl -X POST https://your_domain_name/log-ip `. You will see a JSON response showing your client IP. This will also be recorded in DB.

 ----------------------------------------------------------

 To test database replication, first check the **Primary** db
 
 ```
 pushkar [ ~/client-ip-logger-to-mysql-db-aks ]$ kubectl exec -it -n db mysql-cluster-primary-0 -- bash
 Defaulted container "mysql" out of: mysql, preserve-logs-symlinks (init), volume-permissions (init)
 I have no name!@mysql-cluster-primary-0:/$ mysql -uroot -p
 Enter password: 
 Welcome to the MySQL monitor.  Commands end with ; or \g.
 Your MySQL connection id is 155
 Server version: 9.3.0 Source distribution
 
 Copyright (c) 2000, 2025, Oracle and/or its affiliates.
 
 Oracle is a registered trademark of Oracle Corporation and/or its
 affiliates. Other names may be trademarks of their respective
 owners.
 
 Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
 
 mysql> use flaskdb;
 Reading table information for completion of table and column names
 You can turn off this feature to get a quicker startup with -A
 
 Database changed
 mysql> select * from client_ips;
 +----+----------------+
 | id | ip_address     |
 +----+----------------+
 |  1 | 94.203.158.127 |
 |  2 | 104.28.194.97  |
 |  3 | 15.204.238.116 |
 |  4 | 104.28.226.98  |
 +----+----------------+
 4 rows in set (0.001 sec)
 ```

 Then check the **Secondary** db

 ```
 pushkar [ ~/client-ip-logger-to-mysql-db-aks ]$ kubectl exec -it -n db mysql-cluster-secondary-0 -- bash
 Defaulted container "mysql" out of: mysql, preserve-logs-symlinks (init), volume-permissions (init)
 I have no name!@mysql-cluster-secondary-0:/$ mysql -uroot -p
 Enter password: 
 Welcome to the MySQL monitor.  Commands end with ; or \g.
 Your MySQL connection id is 155
 Server version: 9.3.0 Source distribution
 
 Copyright (c) 2000, 2025, Oracle and/or its affiliates.
 
 Oracle is a registered trademark of Oracle Corporation and/or its
 affiliates. Other names may be trademarks of their respective
 owners.
 
 Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
 
 mysql> use flaskdb;
 Reading table information for completion of table and column names
 You can turn off this feature to get a quicker startup with -A
 
 Database changed
 mysql> select * from client_ips;
 +----+----------------+
 | id | ip_address     |
 +----+----------------+
 |  1 | 94.203.158.127 |
 |  2 | 104.28.194.97  |
 |  3 | 15.204.238.116 |
 |  4 | 104.28.226.98  |
 +----+----------------+
 4 rows in set (0.001 sec)
 ```

---------------------------------------------------------------------

Instructions to deploy **Client IP Logger in MySQL DB** on Azure Kubernetes Service using your own nginx ingress
  1. Deploy AKS cluster through Azure portal.
  2. Deploy **MySQL Replication Cluster** ( 1 primary, 2 secondary ) helm chart. MySQL will be deployed as a **StatefulSet**. Set the **auth.rootPassword** & **auth.replicationPassword** of your choice.
     
     ```
     helm repo add bitnami https://charts.bitnami.com/bitnami
     helm repo update
     
     helm install mysql-cluster bitnami/mysql \
        --namespace db \
        --create-namespace \
        --set architecture=replication \
        --set auth.rootPassword= \
        --set auth.replicationPassword= \
        --set primary.persistence.enabled=true \
        --set primary.persistence.size=10Gi \
        --set primary.persistence.storageClass=managed-csi \
        --set secondary.replicaCount=2 \
        --set secondary.persistence.enabled=true \
        --set secondary.persistence.size=10Gi \
        --set secondary.persistence.storageClass=managed-csi \
        --set volumePermissions.enabled=true
     ```
  3. Create the Table in MySQL which will store the Client IP

     ```
     kubectl exec -it -n db mysql-cluster-primary-0 -- bash
     mysql -uroot -p ( Enter root user password when prompted )

     CREATE DATABASE IF NOT EXISTS flaskdb;
     USE flaskdb;

     CREATE TABLE IF NOT EXISTS client_ips ( id INT AUTO_INCREMENT PRIMARY KEY, ip_address VARCHAR(45) );
     ```
  4. Create a namespace **web** where the application will be deployed.
  5. Put the base64 encoded value of the **auth.rootPassword** set above in **mysql-secret.yaml** & then create the **Secret**.
     
     ```
     kubectl -n web apply -f mysql-secret.yaml
     ```
  6. Create the **ConfigMap** defined in **mysql-configmap.yaml**. The ConfigMap has the values for database host, name & user.
     
     ```
     kubectl -n web apply -f mysql-configmap.yaml
     ```
  7. Create the **Deployment** & **Service**. 

     ```
     kubectl -n web apply -f flask-deployment.yaml -f flask-service.yaml
     ```
  8. Deploy Nginx Ingress Controller by running below commands.

     ```
     helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
     helm repo update
     ```
     ```
     helm install nginx-ingress ingress-nginx/ingress-nginx \
     --set controller.service.externalTrafficPolicy=Local \
     --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"="/" \
     --set controller.service.enableHttps=true
     ```
  9. Run the command ` kubectl get svc nginx-ingress-ingress-nginx-controller ` to confirm if a **LoadBalancer** IP has been provisioned.

     ```
     pushkar [ ~ ]$ kubectl get svc nginx-ingress-ingress-nginx-controller
     NAME                                     TYPE           CLUSTER-IP    EXTERNAL-IP    PORT(S)                      AGE
     nginx-ingress-ingress-nginx-controller   LoadBalancer   10.0.58.180   20.174.45.64   80:32767/TCP,443:30598/TCP   110s
     ```
 10. Create a tls secret named ` cert-tls ` which has the domain's certificate & private key by running below command. The domain's .crt & .key file should already be present.

     ```
     kubectl -n web create secret tls cert-tls --cert=domain_name.crt --key=domain_name.key
     ```
 11. Put the FQDN for which the secret has been created in ` nginx-ingress.yml ` file and then run the command ` kubectl -n web apply -f nginx-ingress.yml `.
 12. Access the app using ` curl -X POST https://your_domain_name/log-ip `. You will see a JSON response showing your client IP. This will also be recorded in DB.

 ----------------------------------------------------------

 To test database replication, first check the **Primary** db
 
 ```
 pushkar [ ~/client-ip-logger-to-mysql-db-aks ]$ kubectl exec -it -n db mysql-cluster-primary-0 -- bash
 Defaulted container "mysql" out of: mysql, preserve-logs-symlinks (init), volume-permissions (init)
 I have no name!@mysql-cluster-primary-0:/$ mysql -uroot -p
 Enter password: 
 Welcome to the MySQL monitor.  Commands end with ; or \g.
 Your MySQL connection id is 155
 Server version: 9.3.0 Source distribution
 
 Copyright (c) 2000, 2025, Oracle and/or its affiliates.
 
 Oracle is a registered trademark of Oracle Corporation and/or its
 affiliates. Other names may be trademarks of their respective
 owners.
 
 Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
 
 mysql> use flaskdb;
 Reading table information for completion of table and column names
 You can turn off this feature to get a quicker startup with -A
 
 Database changed
 mysql> select * from client_ips;
 +----+----------------+
 | id | ip_address     |
 +----+----------------+
 |  1 | 94.203.158.127 |
 |  2 | 104.28.194.97  |
 |  3 | 15.204.238.116 |
 |  4 | 104.28.226.98  |
 +----+----------------+
 4 rows in set (0.001 sec)
 ```

 Then check the **Secondary** db

 ```
 pushkar [ ~/client-ip-logger-to-mysql-db-aks ]$ kubectl exec -it -n db mysql-cluster-secondary-0 -- bash
 Defaulted container "mysql" out of: mysql, preserve-logs-symlinks (init), volume-permissions (init)
 I have no name!@mysql-cluster-secondary-0:/$ mysql -uroot -p
 Enter password: 
 Welcome to the MySQL monitor.  Commands end with ; or \g.
 Your MySQL connection id is 155
 Server version: 9.3.0 Source distribution
 
 Copyright (c) 2000, 2025, Oracle and/or its affiliates.
 
 Oracle is a registered trademark of Oracle Corporation and/or its
 affiliates. Other names may be trademarks of their respective
 owners.
 
 Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
 
 mysql> use flaskdb;
 Reading table information for completion of table and column names
 You can turn off this feature to get a quicker startup with -A
 
 Database changed
 mysql> select * from client_ips;
 +----+----------------+
 | id | ip_address     |
 +----+----------------+
 |  1 | 94.203.158.127 |
 |  2 | 104.28.194.97  |
 |  3 | 15.204.238.116 |
 |  4 | 104.28.226.98  |
 +----+----------------+
 4 rows in set (0.001 sec)
 ```

------------------------------------------------------------

**Helm**
To install this app using Helm using the default **App Routing** add on, perform below steps
  1. Deploy AKS cluster through Azure portal.
  2. Deploy **MySQL Replication Cluster** ( 1 primary, 2 secondary ) helm chart. MySQL will be deployed as a **StatefulSet**. Set the **auth.rootPassword** & **auth.replicationPassword** of your choice.
     
     ```
     helm repo add bitnami https://charts.bitnami.com/bitnami
     helm repo update
     
     helm install mysql-cluster bitnami/mysql \
        --namespace db \
        --create-namespace \
        --set architecture=replication \
        --set auth.rootPassword= \
        --set auth.replicationPassword= \
        --set primary.persistence.enabled=true \
        --set primary.persistence.size=10Gi \
        --set primary.persistence.storageClass=managed-csi \
        --set secondary.replicaCount=2 \
        --set secondary.persistence.enabled=true \
        --set secondary.persistence.size=10Gi \
        --set secondary.persistence.storageClass=managed-csi \
        --set volumePermissions.enabled=true
     ```
  3. Create the Table in MySQL which will store the Client IP

     ```
     kubectl exec -it -n db mysql-cluster-primary-0 -- bash
     mysql -uroot -p ( Enter root user password when prompted )

     CREATE DATABASE IF NOT EXISTS flaskdb;
     USE flaskdb;

     CREATE TABLE IF NOT EXISTS client_ips ( id INT AUTO_INCREMENT PRIMARY KEY, ip_address VARCHAR(45) );
     ```
  4. Run the command to install **Client IP Logger**

     ```
     helm install client-ip-logger ./helm --namespace web --create-namespace --set ingressClassName="web-app-routing" --set domain_name=your_preferred_fqdn --set-file tlsSecret.cert=domain_name.crt --set-file tlsSecret.key=domain_name.key --set mysql_root_password=root_password_set_above
     ```
  5. Run the command ` kubectl -n app-routing-system get svc nginx ` to confirm if a **LoadBalancer** IP has been provisioned.

     ```
     pushkar [ ~ ]$ kubectl -n app-routing-system get svc nginx
     NAME    TYPE           CLUSTER-IP    EXTERNAL-IP    PORT(S)                      AGE
     nginx   LoadBalancer   10.0.58.180   20.174.45.64   80:32767/TCP,443:30598/TCP   110s
     ```
  6. Run `kubectl -n web get ingress` to retrieve the IP. This may take some time to match with the **LoadBalancer** IP above. Point the domain name in your registrar to the IP address.
  7. Access the app using ` curl -X POST https://your_domain_name/log-ip `. You will see a JSON response showing your client IP. This will also be recorded in DB.
  8. Uninstall the app using `helm uninstall client-ip-logger --namespace web`.

  ------------------------------------------------------------

  To test database replication, first check the **Primary** db
 
  ```
  pushkar [ ~/client-ip-logger-to-mysql-db-aks ]$ kubectl exec -it -n db mysql-cluster-primary-0 -- bash
  Defaulted container "mysql" out of: mysql, preserve-logs-symlinks (init), volume-permissions (init)
  I have no name!@mysql-cluster-primary-0:/$ mysql -uroot -p
  Enter password: 
  Welcome to the MySQL monitor.  Commands end with ; or \g.
  Your MySQL connection id is 155
  Server version: 9.3.0 Source distribution
  
  Copyright (c) 2000, 2025, Oracle and/or its affiliates.
  
  Oracle is a registered trademark of Oracle Corporation and/or its
  affiliates. Other names may be trademarks of their respective
  owners.
  
  Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
  
  mysql> use flaskdb;
  Reading table information for completion of table and column names
  You can turn off this feature to get a quicker startup with -A
  
  Database changed
  mysql> select * from client_ips;
  +----+----------------+
  | id | ip_address     |
  +----+----------------+
  |  1 | 94.203.158.127 |
  |  2 | 104.28.194.97  |
  |  3 | 15.204.238.116 |
  |  4 | 104.28.226.98  |
  +----+----------------+
  4 rows in set (0.001 sec)
  ```
 
  Then check the **Secondary** db
 
  ```
  pushkar [ ~/client-ip-logger-to-mysql-db-aks ]$ kubectl exec -it -n db mysql-cluster-secondary-0 -- bash
  Defaulted container "mysql" out of: mysql, preserve-logs-symlinks (init), volume-permissions (init)
  I have no name!@mysql-cluster-secondary-0:/$ mysql -uroot -p
  Enter password: 
  Welcome to the MySQL monitor.  Commands end with ; or \g.
  Your MySQL connection id is 155
  Server version: 9.3.0 Source distribution
  
  Copyright (c) 2000, 2025, Oracle and/or its affiliates.
  
  Oracle is a registered trademark of Oracle Corporation and/or its
  affiliates. Other names may be trademarks of their respective
  owners.
  
  Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
  
  mysql> use flaskdb;
  Reading table information for completion of table and column names
  You can turn off this feature to get a quicker startup with -A
  
  Database changed
  mysql> select * from client_ips;
  +----+----------------+
  | id | ip_address     |
  +----+----------------+
  |  1 | 94.203.158.127 |
  |  2 | 104.28.194.97  |
  |  3 | 15.204.238.116 |
  |  4 | 104.28.226.98  |
  +----+----------------+
  4 rows in set (0.001 sec)
  ```

-----------------------------

**Helm**
To install this app using Helm using your own nginx ingress, perform below steps
  1. Deploy AKS cluster through Azure portal.
  2. Deploy **MySQL Replication Cluster** ( 1 primary, 2 secondary ) helm chart. MySQL will be deployed as a **StatefulSet**. Set the **auth.rootPassword** & **auth.replicationPassword** of your choice.
     
     ```
     helm repo add bitnami https://charts.bitnami.com/bitnami
     helm repo update
     
     helm install mysql-cluster bitnami/mysql \
        --namespace db \
        --create-namespace \
        --set architecture=replication \
        --set auth.rootPassword= \
        --set auth.replicationPassword= \
        --set primary.persistence.enabled=true \
        --set primary.persistence.size=10Gi \
        --set primary.persistence.storageClass=managed-csi \
        --set secondary.replicaCount=2 \
        --set secondary.persistence.enabled=true \
        --set secondary.persistence.size=10Gi \
        --set secondary.persistence.storageClass=managed-csi \
        --set volumePermissions.enabled=true
     ```
  3. Create the Table in MySQL which will store the Client IP

     ```
     kubectl exec -it -n db mysql-cluster-primary-0 -- bash
     mysql -uroot -p ( Enter root user password when prompted )

     CREATE DATABASE IF NOT EXISTS flaskdb;
     USE flaskdb;

     CREATE TABLE IF NOT EXISTS client_ips ( id INT AUTO_INCREMENT PRIMARY KEY, ip_address VARCHAR(45) );
     ```
  4. Deploy **Nginx Ingress Controller** by running below commands.
     
     ```
     helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
     helm repo update
     ```
     ```
     helm install nginx-ingress ingress-nginx/ingress-nginx \
     --set controller.service.externalTrafficPolicy=Local \
     --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"="/" \
     --set controller.service.enableHttps=true
     ```
  5. Run the command ` kubectl get svc nginx-ingress-ingress-nginx-controller ` to confirm if a **LoadBalancer** IP has been provisioned.

     ```
     pushkar [ ~ ]$ kubectl get svc nginx-ingress-ingress-nginx-controller
     NAME                                     TYPE           CLUSTER-IP    EXTERNAL-IP    PORT(S)                      AGE
     nginx-ingress-ingress-nginx-controller   LoadBalancer   10.0.58.180   20.174.45.64   80:32767/TCP,443:30598/TCP   110s
     ```
  6. Run the command to install **Client IP Logger**

     ```
     helm install client-ip-logger ./helm --namespace web --create-namespace --set ingressClassName="nginx" --set domain_name=your_preferred_fqdn --set-file tlsSecret.cert=domain_name.crt --set-file tlsSecret.key=domain_name.key --set mysql_root_password=root_password_set_above
     ```
  5. Run `kubectl -n web get ingress` to retrieve the IP. This may take some time to match with the **LoadBalancer** IP above. Point the domain name in your registrar to the IP address.
  6. Access the app using ` curl -X POST https://your_domain_name/log-ip `. You will see a JSON response showing your client IP. This will also be recorded in DB.
  7. Uninstall the app using `helm uninstall client-ip-logger --namespace web`.

  ------------------------------------------------------------

  To test database replication, first check the **Primary** db
 
  ```
  pushkar [ ~/client-ip-logger-to-mysql-db-aks ]$ kubectl exec -it -n db mysql-cluster-primary-0 -- bash
  Defaulted container "mysql" out of: mysql, preserve-logs-symlinks (init), volume-permissions (init)
  I have no name!@mysql-cluster-primary-0:/$ mysql -uroot -p
  Enter password: 
  Welcome to the MySQL monitor.  Commands end with ; or \g.
  Your MySQL connection id is 155
  Server version: 9.3.0 Source distribution
  
  Copyright (c) 2000, 2025, Oracle and/or its affiliates.
  
  Oracle is a registered trademark of Oracle Corporation and/or its
  affiliates. Other names may be trademarks of their respective
  owners.
  
  Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
  
  mysql> use flaskdb;
  Reading table information for completion of table and column names
  You can turn off this feature to get a quicker startup with -A
  
  Database changed
  mysql> select * from client_ips;
  +----+----------------+
  | id | ip_address     |
  +----+----------------+
  |  1 | 94.203.158.127 |
  |  2 | 104.28.194.97  |
  |  3 | 15.204.238.116 |
  |  4 | 104.28.226.98  |
  +----+----------------+
  4 rows in set (0.001 sec)
  ```
 
  Then check the **Secondary** db
 
  ```
  pushkar [ ~/client-ip-logger-to-mysql-db-aks ]$ kubectl exec -it -n db mysql-cluster-secondary-0 -- bash
  Defaulted container "mysql" out of: mysql, preserve-logs-symlinks (init), volume-permissions (init)
  I have no name!@mysql-cluster-secondary-0:/$ mysql -uroot -p
  Enter password: 
  Welcome to the MySQL monitor.  Commands end with ; or \g.
  Your MySQL connection id is 155
  Server version: 9.3.0 Source distribution
  
  Copyright (c) 2000, 2025, Oracle and/or its affiliates.
  
  Oracle is a registered trademark of Oracle Corporation and/or its
  affiliates. Other names may be trademarks of their respective
  owners.
  
  Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
  
  mysql> use flaskdb;
  Reading table information for completion of table and column names
  You can turn off this feature to get a quicker startup with -A
  
  Database changed
  mysql> select * from client_ips;
  +----+----------------+
  | id | ip_address     |
  +----+----------------+
  |  1 | 94.203.158.127 |
  |  2 | 104.28.194.97  |
  |  3 | 15.204.238.116 |
  |  4 | 104.28.226.98  |
  +----+----------------+
  4 rows in set (0.001 sec)
  ```

-----------------------------
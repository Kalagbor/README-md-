# README-md-
#DevOps take-home test

# Complete Manifest in single K8s yaml we defined Deployment & Service for SpringApp & PVC, (with default  StorageClass), HPA, ReplicaSet & Service For Mongo db.
# In this manifest file container1 is the springapp that returns users from mongo database
# Deploment manifest file for container1

---
apiVersion: v1
kind: ConfigMap
metadata: 
  name: springappconfigmap
data: 
  mongousername: admin*user
  mongopassword: adminuser*123
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springappdeployment
spec:
  replicas: 2
  strategy: 
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 2
      mazsurge: 1
  minReadySeconds: 30
  selector:
    matchLabels:
      app: springapp
  template:
    metadata:
      name: springapppod
      labels:
        app: springapp
    spec:
      containers:
      - name: springappcontainer # container1
        image: kalaski/spring-boot-mongo
        ports:
        - containerPort: 8080
        env:
        - name: MONGO_DB_USERNAME
          valueFrom:
            configMapKeyRef:
              name: mongousername
        - name: MONGO_DB_HOSTNAME
          valueFrom:
            configMapKeyRef: 
              name: mongopassword
          requests:
            cpu: 200m
            memory: 64Mi
          limits:
            cpu: 500m
            memory: 1Gi
        livelinessProbe:
          httpGet:
            path: /spring-boot-app
            part: 8080
        readinessProbe:
          httpGet:
            path: /spring-boot-app
            part: 8080
            
---
apiVersion: v1
kind: Service
metadata:
  name: springapp
spec:
  selector:
    app: springapp
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodbpvc 
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: mongodbrs
spec:
  selector:
    matchLabels:
      app: mongodb
  template:
     metadata:
       name: mongodbpod
       labels:
         app: mongodb
     spec:
       volumes:
       - name: pvc
         persistentVolumeClaim:
           claimName: mongodbpvc     
       containers:
       - name: mongodbcontainer
         image: mongo
         ports:
         - containerPort: 27017
         env:
         - name: MONGO_INITDB_ROOT_USERNAME
           value: devdb
         - name: MONGO_INITDB_ROOT_PASSWORD
           value: devdb@123
         volumeMounts:
         - name: pvc
           mountPath: /data/db   
---
apiVersion: v1
kind: Service
metadata:
  name: mongo
spec:
  type: ClusterIP
  selector:
    app: mongodb
  ports:
  - port: 27017
    targetPort: 27017
 ---
 apiVersion: autoscaling/v2beta1
 kind: HorizontalPodAutoscaler
 metadata:
   name: springappdeployment
 spec:
   scaleTargetRef:
     apiVersion: apps/v1
     kind: Deployment
   miniReplicas: 2
   maxreplicas: 10
   metrics:
     - resource:
         name: cpu
         targetAverageUtilization: 70
       type: Resource
 
=================================================================================================================================

# Complete Manifest in single K8s yaml we defined Deployment & Service for mysql & PVC, (with default  StorageClass) and HPA.
# In this manifest file container2 is named dbapp that returns users from mysql database
# Deploment manifest file for container2

--
kind: ConfigMap 
apiVersion: v1 
metadata:
  name: mysql-configmap 
data:
  # Configuration values can be set as key-value properties
  # configMap for the dev env
  password: devdb@123
  
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dbappdeployment # container2 deployment
  labels:
    app: dbapp
spec:
  replicas: 2
  strategy: 
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 2
      mazsurge: 1
  minReadySeconds: 30
  selector:
    matchLabels:
      app: dbapp
      tier: mysql
  template:
    metadata:
      labels:
        app: dbapp
        tier: mysql
    spec:
      containers:
        - image: mysql:5.7
          imagePullPolicy: Always
          name: mysql
          env:
          - name: MYSQL_ROOT_PASSWORD
            valueFrom:
               configMapKeyRef:
                  name: mysql-configmap
                  key: password
          ports:
            - containerPort: 3306
              name: mysql
          resources:
            requests:
              cpu: 200m
              memory: 64Mi
            limits:
              cpu: 500m
              memory: 1Gi
          livelinessProbe:
            httpGet:
              path: /spring-boot-app
              part: 8080
          readinessProbe:
            httpGet:
              path: /spring-boot-app
              part: 8080
          volumeMounts:
            - name: mysql-persistent-storage
              mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        hostPath:
          path: /tmp/mysqlbackup
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysqlpvc 
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Service
metadata:
  name: dbapp
  labels:
    app: dbapp
spec:
  ports:
    - port: 3306
  selector:
    app: dbapp
    tier: mysql
---
 apiVersion: autoscaling/v2beta1
 kind: HorizontalPodAutoscaler
 metadata:
   name: dbappdeployment
 spec:
   scaleTargetRef:
     apiVersion: apps/v1
     kind: Deployment
   miniReplicas: 2
   maxreplicas: 10
   metrics:
     - resource:
         name: cpu
         targetAverageUtilization: 70
       type: Resource
       
       
 Question4
 To ensure that my dev team only perform certain roles. As the Kubernetes administratior, I will create a role and bind all the menbers of the dev team to that role using RBAC. This can be executed using the manifest file;
 
 #RBAC to assign privilleges to members of the dev team

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: deployment
rules:
- apiGroups: ["apps"] # "" indicates the core API group
  resources: ["deployment"]
  verbs: ["create"]
  
 ---
 
 apiVersion: rbac.authorization.k8s.io/v1
# This role binding allows "devteam" to deploy and rollback in the "dev" namespace.
kind: RoleBinding
metadata:
  name: deploy-pods
  namespace: dev
subjects:
# You can specify more than one "subject"
- kind: user
  name: deploymentteam # "name" is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  # "roleRef" specifies the binding to a Role / ClusterRole
  kind: Role #this must be Role or ClusterRole
  name: deployment # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
================================================================================================================================================================

Bonus 1
1) To apply your configMap on multiple environment; the developer much not hardcode the pom.xml file assuming his application is java based app, so with this
the username and the password of the environmental variables of the configMap can changed and easily applied in the staging and production environment; asuming container2 was deployed to the dev environment, in the staging and production env the configMap will be written as;

--
kind: ConfigMap 
apiVersion: v1 
metadata:
  name: mysql-configmap 
data:
  # Configuration values can be set as key-value properties
  # configMap for the staging env
  password: stagedb@123
  
--
kind: ConfigMap 
apiVersion: v1 
metadata:
  name: mysql-configmap 
data:
  # Configuration values can be set as key-value properties
  # configMap for the production env
  password: proddb@123
These changes will be applied will deploying the containers in the env variable of the database in the staging and production environment.

Bonus2
Auto-scaling a deployment based on network latency involves monitoring the latency metrics and adjusting the number of instances of the deployment dynamically to maintain optimal performance. To autoscale base on network latency we can do the following; Set Thresholds to determine the threshold values for latency metrics beyond which the deployment needs to scale up or down. For instance, you may set a threshold for the 99th percentile latency such that if it goes above a certain value for a specific duration, you'll need to scale up the deployment. Also, once the threshold value is reached, you can automate the deployment to scale up or down automatically. If the latency is high, the deployment can scale up by adding more instances to handle the increased traffic. Conversely, if the latency is low, the deployment can scale down by removing instances to save resources.

# manifest files needed to realized the project

for jenkins to connect with the cluster for any deployment to be made jenkins need authentication and authorization this will be achieve by creating the following 
manifest files:
- ### Create serviveaccount for Jenkins
- ### Create Role for jenkins
- ### Assign Role to serviceaccount (RoleBinding)
- ### Create a secret for jenkins to authenticate to the cluster
- ### create a manifest file for the deployment
- ### Install kubectl on jenkins virtual machine 

1. create a serviceaccount **svc.yaml**

  ```svc.yaml
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: jenkins
    namespace: webapp
  ```

  Create a namespace:

  ```
  kubectl create ns webapp
  ```
  ```
  kubectl apply -f svc.yaml
  ```
2. Create a Role **role.yaml**

```role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: webapp
rules:
  - apiGroups:
        - ""
        - apps
        - autoscaling
        - batch
        - extensions
        - policy
        - rbac.authorization.k8s.io
    resources:
      - pods
      - secrets
      - componentstatuses
      - configmaps
      - daemonsets
      - deployments
      - events
      - endpoints
      - horizontalpodautoscalers
      - ingress
      - jobs
      - limitranges
      - namespaces
      - nodes
      - pods
      - persistentvolumes
      - persistentvolumeclaims
      - resourcequotas
      - replicasets
      - replicationcontrollers
      - serviceaccounts
      - services
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

```
kubectl apply -f role.yaml
```
3. ## Assign role to the serviceaccount (roleBinding)

```bind.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-rolebinding
  namespace: webapp
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: app-role
subjects:
- namespace: webapps
  kind: ServiceAccount
  name: jenkins
```

```
kubectl apply -f bind.yaml
```

4. ## Create a token to use for authentication by creating a secrete

```secrete.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecrete
  annotations:
    kubernetes.io/service-account.name: jenkins
type: kubernetes.io/service-account-token
```
because we didnot provide the namespace in the manifest file for the secrete we need to pass the neame space when apply the file 

## the following should be added to the Jenkins controller to generate our pipeline syntax to use in our Jenkinsfile and the deployment stage of the jenkinsfile:

| name             | value                    |
| ---              | ---                      |
| token            | secrete text             |
| ---              | ---                      |
| cluster name     | kubernetes               |
| --               | ---                      |
| cluster endpoint |  https://ip address:6443 |
| namespace        |  webapp                  |
|   ---            | ---                      |
```
kubectl apply -f secrete.yaml -n webapp
```
5. ## run the command to use the token created by the secrete file which we need to provide as credential to jenkins so we can authenticate with k8s

```
kubectl describe secrets mysecrete -n webapp
```

> Use the output of the _kubectl describe command_ to get the token and copy the token to the ***jenkins controller*** and go to the credential section choose
> crendential of type **_secret text_** and paste the secrete.

6.  To get the **_k8s server endpoint_** and **_cluster name_**. in our home directory do the following:

```
cd ~/.kube
```

```
ls 
```

```
cat config
```

the we can verify the **_cluster name_** and the **_server endpoint_** to use in our Jenkis controller to generate our jenkinsfile at the pipeline syntax

7. ## create a manifest file for the deployment of our application
```deployment-service.yaml
apiVersion: apps/v1
kind: Deployment 
metadata:
  name: java-maven-app-deployment
spec:
  selector:
    matchLabels:
      app: java-maven-app
  replicas: 2 # Number of replicas that will be created for this deployment
  template:
    metadata:
      labels:
        app: java-maven-app
    spec:
      containers:
        - name: java-maven-app
          image: juliet237/java-maven-app:1.0 # this image will be used by containers in the cluster
          imagePullPolicy: Always
          ports:
            - containerPort: 8080 # The port that the container is running on in the cluster


---

apiVersion: v1 
kind: Service 
metadata: 
  name: java-maven-app-svc
spec:
  selector:
    app: java-maven-app
  ports:
    - protocol: "TCP"
      port: 8080 # the service is running on this port in the cluster
      targetPort: 8080 # The port exposed by the service
  type: LoadBalancer # type of the service.
```



9. ## Installation of kubectl on Jenkins container since jenkins is running as a Docker Container to permit jenkins to run kubectl command

## Enter inside the container as root user

```
kubectl exec -u 0 -it <container id> /bin/bash
```
## Inside the container run the following command

```
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
```

```
chmod +x ./kubectl
```

```
mv ./kubectl /usr/local/bin
```

```
kubectl version --short --client
```
---
## Installation of Trivy on Jenkins server inside jenkins container since Jenkins is running as a Docker Container

## Enter the container as a root user

```
docker exec -u 0 -it <container id> /bin/bash
```

replace the container id with your actual container id

## modify the permission (read write permission ) of the docker daemon for regular user (Jenkins) to run docker command

```
chmod 666 /var/run/docker.sock
```

Run the following command inside the container as root user

```
apt-get install wget apt-transport-https gnupg lsb-release
```

```
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | tee /usr/share/keyrings/trivy.gpg > /dev/null
```

If the above command does not succeed to download the package then install wget

```
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | tee -a /etc/apt/sources.list.d/trivy.list
```

```
apt-get update
```

```
apt-get install trivy -y
```


```
trivy --version
```

## To scan a filesystem run the command below:

```
trivy fs .
```

## To scan a filesystem an export the report to a particular format

```
trivy fs --format table -o trivy-fs-report.html .
```

## To scan an image

```
trivy image <image_name>
```











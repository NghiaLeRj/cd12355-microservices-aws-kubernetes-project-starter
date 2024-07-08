# I. Getting start
### 1. Configure git: 
```sh
git config --global user.email "nghia.le.rj@renesas.com "
git config --global user.name "nghialerj"
```

### 2. Configure aws: 
- Installing AWS CLI
- Confirm the innstallation: `aws --version`
- To configure AWS: `aws configure`, then fill in the <YOUR_AWS_ACCESS_KEY>, <YOUR_AWS_SECRET_KEY>, region and format type.
- To add session token: `aws configure set aws_session_token <YOUR_AWS_SESSION_TOKEN>`
- To check IAM permission: `aws sts get-caller-identity`

### 3. Create EKS (Elastic Kubernetes Service)
- Create your EKS cluster by command:
```sh
eksctl create cluster --name my-cluster --region us-east-1 --nodegroup-name my-nodes --node-type t3.small --nodes 1 --nodes-min 1 --nodes-max 2

# Where:
#    --name: Your cluster name
#    --region: Your region
#    --nodegroup-name: Your nodegroup name
#    --node-type: Your node configuration
#    --nodes: Number of operation node
#    --nodes-min: Limit the minimum of node by 1
#    --nodes-max: Limit the maximum of node by 2
```

- Update your kubeconfig:
```sh
aws eks --region us-east-1 update-kubeconfig --name my-cluster
```
- Verify the context name:
```sh
kubectl config current-context

#The output will  be: arn:aws:eks:us-east-1:935844883043:cluster/my-project-cluster
```
- To view the kubeconfig:
```sh 
kubectl config view
```

- Note: To delete your cluster in case not use, type this command:
```sh
eksctl delete cluster --name my-cluster --region us-east-1
```

# II. Configure database
- Confirm you are connected to k8s:
```sh
kubectl get namespace
```

- Create PersistentVolumeClaim: create a file named as `pvc.yaml` on your local machine
```yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgresql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1G
  volumeName: my-manual-pv
  storageClassName: gp2
```
- Create PersistentVolume: create a file named as `pv.yaml` on your local machine
```yml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-manual-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: gp2
  hostPath:
    path: "/mnt/data"
```
- Create Postgres Deployment:

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgresql
spec:
  selector:
    matchLabels:
      app: postgresql
  template:
    metadata:
      labels:
        app: postgresql
    spec:
      containers:
      - name: postgresql
        image: postgres:latest
        env:
        - name: POSTGRES_DB
          value: mydatabase
        - name: POSTGRES_USER
          value: <your_user_name>
        - name: POSTGRES_PASSWORD
          value: <your_password>
        ports:
        - containerPort: 5432
        volumeMounts:
        - mountPath: /var/lib/postgresql/data
          name: postgresql-storage
      volumes:
      - name: postgresql-storage
        persistentVolumeClaim:
          claimName: postgresql-pvc
```
- Apply the yaml configuration:
```sh
kubectl apply -f pvc.yaml
kubectl apply -f pv.yaml
kubectl apply -f postgresql-deployment.yaml
```
- Connecting via Port Forwarding:
    - Create new file named as `postgresql-service.yaml` on your local machine
    ```yml
    apiVersion: v1
    kind: Service
    metadata:
        name: postgresql-service
    spec:
        ports:
        - port: 5432
            targetPort: 5432
        selector:
            app: postgresql
    ```
    - Apply the yaml file to create service:
    ```sh
    kubectl apply -f postgresql-service.yaml
    ```
    - Run the following command to forward the port:
    ```sh
    # List the services
    kubectl get svc

    # Set up port-forwarding to `postgresql-service`
    kubectl port-forward service/postgresql-service 5433:5432 &
    ```
- Run seed files:
    - Install `psql` on your machine:
    ```sh
    apt update
    apt install postgresql postgresql-contrib
    ```
    - Run the seed files inside `/db` folder:
    ```sh
    export DB_PASSWORD=<POSTGRES_PASSWORD>
    PGPASSWORD="$DB_PASSWORD" psql --host 127.0.0.1 -U <POSTGRES_USER> -d mydatabase -p 5433 -f 1_create_tables.sq
    PGPASSWORD="$DB_PASSWORD" psql --host 127.0.0.1 -U <POSTGRES_USER> -d mydatabase -p 5433 -f 2_seed_users.sql
    PGPASSWORD="$DB_PASSWORD" psql --host 127.0.0.1 -U <POSTGRES_USER> -d mydatabase -p 5433 -f 3_seed_tokens.sql
    ```
- Check the tables to confirm the seed file has been applied successfully:
```sh
PGPASSWORD="$DB_PASSWORD" psql --host 127.0.0.1 -U <POSTGRES_USER> -d mydatabase -p 5433

#You will turn to database, then run this command:
select *from users;
select* from tokens;

#To exit database, type this:
\q
```
- Note: To close your forward port, usee this command:
```sh
ps aux | grep 'kubectl port-forward' | grep -v grep | awk '{print $2}' | xargs -r kill
```
# III. Build the analytics application locally
- Install dependencies:
```sh
# Update the local package index with the latest packages from the repositories
apt update

# Install a couple of packages to successfully install postgresql server locally
apt install build-essential libpq-dev

# Update python modules to successfully build the required modules
pip install --upgrade pip setuptools wheel

#Install the module list
pip install -r requirements.txt
```

- Run the application:
```sh
export DB_USERNAME=<POSTGRES_USER>
export DB_PASSWORD=<POSTGRES_PASSWORD>
export DB_HOST=127.0.0.1 
export DB_PORT=5433
export DB_NAME=mydatabase

python app.py
```

- Verify the application by typing below commands:
```sh
curl http://127.0.0.1:5153/api/reports/daily_usage
curl http://127.0.0.1:5153/api/reports/user_visits
```

# IV. Deploy the analytics application
- Dockerize the Application:
    - Update Docker Version:
    ```sh
    apt update
    apt install docker-ce docker-ce-cli containerd.io
    ```
    - Build and Run a Docker Image:
        - Dockerfile content:
        ```docker
        FROM python:3.10-slim-buster
        COPY ./ /app
        WORKDIR /app

        RUN apt update -y

        RUN apt install build-essential -y libpq-dev

        RUN pip install --upgrade pip setuptools wheel

        RUN pip install -r $PWD/requirements.txt

        EXPOSE 5153
        EXPOSE 5433

        #Export variable
        ENV DB_USERNAME <POSTGRES_USER>
        ENV DB_PASSWORD $POSTGRESQL_PW
        ENV DB_HOST host.docker.internal
        ENV DB_PORT 5433
        ENV DB_NAME mydatabase

        #Run application
        CMD [ "python", "app.py"]

        #Command run
        #docker run -it  -p 5433:5433 -p 5153:5153 -e POSTGRESQL_PW=<password> test-coworking-analytics
        ```
        - Build the docker:
        ```sh
        docker build -t test-coworking-analytics .
        ```
        - Verify the docker image:
        ```sh
        docker run -it  -p 5433:5433 -p 5153:5153 -e POSTGRESQL_PW=<password> test-coworking-analytics
        
        #Confirm the application by command:
        curl http://127.0.0.1:5153/api/reports/daily_usage
        curl http://127.0.0.1:5153/api/reports/user_visits
        ```
- Set up Continuous Integration with CodeBuild:
    - Create an Amazon ECR repository on your AWS console.
    - Create a file `buildspec.yaml` with the following content:
    ```yml
    version: 0.2
    phases:
    pre_build:
        commands:
        - echo Logging into ECR
        - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
    build:
        commands:
        - echo Starting build at `date`
        - echo Building the Docker image...          
        - docker build -t $IMAGE_REPO_NAME:$CODEBUILD_BUILD_NUMBER ./analytics/.
        - docker tag $IMAGE_REPO_NAME:$CODEBUILD_BUILD_NUMBER $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$CODEBUILD_BUILD_NUMBER      
    post_build:
        commands:
        - echo Completed build at `date`
        - echo Pushing the Docker image...
        - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$CODEBUILD_BUILD_NUMBER
    ```
    - Create your `CodeBuild` project on AWS and start build.
- Deploy the Application
    - Create configuration map:
        - configmap.yaml:
        ```yml
        apiVersion: v1
        kind: ConfigMap
        metadata:
            name: postgresql-service-postgresql
        data:
            DB_NAME: "mydatabase"
            DB_USERNAME: "nghiale"
            DB_HOST: "postgresql-service"
            DB_PORT: "5432"
        ---
        apiVersion: v1
        kind: Secret
        metadata:
            name: db-secret
        type: Opaque
        data:
            DB_PASSWORD: "bmdoaWFsZXJqMTk5Nw=="
        ```
            - Note: The DB_PASSWORD value should be encoded, for example: `echo -n 'password' | base64`
        - coworking.yaml:
        ```yml
        apiVersion: v1
        kind: Service
        metadata:
            name: coworking
        spec:
            type: LoadBalancer
            selector:
                service: coworking
            ports:
            - name: "5153"
                protocol: TCP
                port: 5153
                targetPort: 5153
        ---
        apiVersion: apps/v1
        kind: Deployment
        metadata:
            name: coworking
            labels:
                name: coworking
        spec:
            replicas: 1
            selector:
                matchLabels:
                service: coworking
            template:
                metadata:
                labels:
                    service: coworking
                spec:
                containers:
                - name: coworking
                    image: <DOCKER_IMAGE_URI_FROM_ECR>
                    imagePullPolicy: IfNotPresent
                    livenessProbe:
                        httpGet:
                            path: /health_check
                            port: 5153
                        initialDelaySeconds: 5
                        timeoutSeconds: 2
                    readinessProbe:
                        httpGet:
                            path: "/readiness_check"
                            port: 5153
                        initialDelaySeconds: 5
                        timeoutSeconds: 5
                    envFrom:
                    - configMapRef:
                        name: postgresql-service-postgresql
                    env:
                    - name: DB_PASSWORD
                        valueFrom:
                            secretKeyRef:
                            name: db-secret
                            key: DB_PASSWORD
                restartPolicy: Always
        ```
    - Apply the new yaml configured file:
    ```sh
    kubectl apply -f ./deployment/
    ```
    - Verify the deployment:
    ```sh
    kubectl get svc    #Get the <EXTERNAL IP> from your service name inside yaml file
    kubectl get deploy #Confirm the deployment is at READY state
    kubectl get pods   #Confirm the pod is at READY state and STATUS is Running

    #Confirm the application:
    curl <EXTERNAL IP>:5153/api/reports/daily_usage
    curl <EXTERNAL IP>:5153/api/reports/user_visits
    ```
# V. Cloud watch logging
- Configure container insight to check the application logs:
```sh
aws iam attach-role-policy --role-name <your-node-role> --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
aws eks create-addon --addon-name amazon-cloudwatch-observability --cluster-name <your-cluster-name>

#Example command:
aws iam attach-role-policy --role-name eks-node-role --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
aws eks create-addon --addon-name amazon-cloudwatch-observability --cluster-name my-project-cluster

#To see the log, run some commands to get response from your application:
curl <EXTERNAL IP>:5153/api/reports/daily_usage
curl <EXTERNAL IP>:5153/api/reports/user_visits
```


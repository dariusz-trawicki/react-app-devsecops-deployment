# Deploying a React app example on Kubernetes using DevSecOps methodology

This project deploys a sample React application on an `Amazon EKS cluster` using `DevSecOps methodology`. It use security tools like `OWASP Dependency Check` and `Trivy`.
It would also be monitoring its `EKS cluster` using monitoring tools such as `Prometheus` and `Grafana`. Most importantly, it would be using `ArgoCD` for deployment.

GitHub repositories (required for the example to work correctly):
- the app: `https://github.com/dariusz-trawicki/react-app-example`
- the terraform code: `https://github.com/dariusz-trawicki/react-app-devsecops-deployment`

## Step 1: Launch an EC2 Instance and install Jenkins, Docker and Trivy

We would be making use of Terraform to launch the EC2 instance. We would be adding a script as userdata for the installation of Jenkins, Trivy and Docker. 

### Create EC2 instance

Update the parameters in `terraform.tfvars` with your own AWS details:
- `server_name` – a custom name for your server (e.g., `jenkins-server`)
- `vpc_id` – the ID of your existing VPC in AWS (e.g., `vpc-0123456789abcdef0`)
- `ami` – the ID of the `Amazon Machine Image` you want to use (e.g., `ami-0a72753edf3e631b7`)
- `key_pair` – the name of your existing EC2 key pair for `SSH access`
- `subnet_id` – the ID of the subnet in which the server will be launched (e.g., `subnet-0123456789abcdef0`)

On the `localhost` run:

```bash
git clone https://github.com/dariusz-trawicki/react-app-devsecops-deployment
cd react-app-devsecops-deployment/jenkins-trivy-server
terraform init
terraform plan
terraform apply
# *** output (example) ***
# ec2_public_ip = "52.59.138.177"
```

## Step 2: Build the Docker image with a React app example (application code from GitHub)

Open: `AWS >  EC2 > Instances` - check the server and click `Connect` and run:

```bash
git clone https://github.com/dariusz-trawicki/react-app-example
cd react-app-example
ls
docker build -t react-app-example .
docker images
# *** output ***
# REPOSITORY   TAG             IMAGE ID        CREATED        SIZE
# react-app-example       latest          af9d40175e3d   3 minutes ago   461MB

docker run -d -p 3000:3000 --name react-app-example react-app-example
```

Open: `http://52.59.138.177:3000`


## Step 2: Access Jenkins at port 8080 and install required plugins

Open: `http://52.59.138.177:8080`

### Jenkins- initial password

In `EC2 CLI`, run:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
# f42154e4c3db4ea9a067511129850397
# OR:
sudo systemctl status jenkins
# *** output ***
# f42154e4c3db4ea9a067511129850397
```

Open: `http://52.59.138.177:8080/manage/pluginManager/available`

Install the following `plugins`:

1. NodeJS 
2. Eclipse Temurin Installer
3. OWASP Dependency Check
4. Docker
5. Docker Commons
6. Docker Pipeline
7. Docker API
8. docker-build-step

## Step 4: Set up OWASP Dependency Check 

1. Go to `Manage Jenkins > Tools` -> `Dependency-Check Installations`
-> Click `Add` and set:
- Name: `OWASP DP-Check`
- check: `Install automatically`
    - from the `Add instalator` list choose: `Install from github.com`
...and do the Step 5 (on the same page):

## Step 5: Set up Docker for Jenkins

1. Go to `Manage Jenkins > Tools` -> `Docker Installations` 
-> Click `Add` and set:
- Name: `docker`
- check: `Install automatically`
    - from the `Add instalator` list choose: `Install from docker.com`

Click `SAVE`.

2. For the docker registry (must have: an account on `Dockerhub`) -  go to `Manage Jenkins > Credentials > System > Global Credentials -> Add redentials`: Set:
- kind: `Add username and password`
- username: `USER_NAME` # replace with real
- password: `PASSWORD`  # replace with real
- ID: `docker-cred`

## Step 6: Create a pipeline in order to build and push the dockerized image securely using multiple security tools

Go to `Dashboard > New Item` -> set Name: e.g. `ReactProject` and choose `Pipeline` and presas `OK`. 
Choose: `Discard old builds` and set:
- Keep # of builds to keep: 2

Use the code below for the `Jenkins pipeline` (`Script` field).

```bash
pipeline {
    agent any
    stages {
        stage('clean workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/dariusz-trawicki/react-app-example'
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'OWASP DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                script {
                    try {
                        sh "trivy fs . > trivyfs.txt" 
                    }catch(Exception e){
                        input(message: "Are you sure to proceed?", ok: "Proceed")
                    }
                }
            }
        }
        stage("Docker Build Image"){
            steps{
                   
                sh "docker build -t react-app-example ."
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image react-app-example > trivyimage.txt"
                script{
                    input(message: "Are you sure to proceed?", ok: "Proceed")
                }
            }
        }
        stage("Docker Push"){
            steps{
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker'){   
                    sh "docker tag react-app-example dariusztrawicki/react-app-example:latest"
                    sh "docker push dariusztrawicki/react-app-example:latest"
                    }
                }
            }
        }
    }
}
```

Press `SAVE` button.

### Run teh pipeline

From the left menu choose `Build Now` link.

## Step 7: Create an EKS Cluster using Terraform

On `localhost` (in folder: `react-app-devsecops-deployment/eks`) run:

```bash
terraform init
terraform plan
terraform apply
```

## Step 8: Deploy Prometheus and Grafana on EKS 

### Install (on the EC2 server)  `kubectl` and `helm` 

```bash
# install kubectl:
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl && \
chmod +x ./kubectl && \
sudo mv ./kubectl /usr/local/bin/kubectl
# test:
kubectl


# install helm:
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
# test
helm version
```

### Access the EKS cluster

In order to access the cluster use (on EC2 server) the command below:

```bash
aws configure
# set the proper values:
# AWS Access Key ID [None]: AKIXXXXXXXXXXXX
# Secret Access Key [None]: aoKcxxxxxxxxxxxxxxxxxxxxxxxxx
# Default region name [None]: eu-central-1

# configure the kubectl to communicate with the EKS cluster
aws eks update-kubeconfig --name react-app-example-cluster --region eu-central-1
kubectl get nodes
# IF: command return error like:
# err="couldn't get current server API group list: the server has asked for the client to provide credentials
# THEN:
aws sts get-caller-identity      
# {
#     "UserId": "AIDXXXXXXXXXXXXX",
#     "Account": "25XXXXXX",
#     "Arn": "arn:aws:iam::25XXXXXX:user/USER_ACCOUNT_NAME"
# }

aws eks create-access-entry \
  --cluster-name react-app-example-cluster \
  --region eu-central-1 \
  --principal-arn arn:aws:iam::25XXXXXX:user/USER_ACCOUNT_NAME

aws eks associate-access-policy \
  --cluster-name react-app-example-cluster \
  --region eu-central-1 \
  --principal-arn arn:aws:iam::25XXXXXX:user/USER_ACCOUNT_NAME \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster
kubectl get nodes
# *** output (example) ***
# NAME                                      STATUS   ROLES    AGE   VERSION
# ip-10-123-3-111.eu-central-1.compute.internal   Ready    <none>   32m   v1.33.0-eks-802817d
```

1. Add the `Helm Stable Charts`

```bash
helm repo add stable https://charts.helm.sh/stable
```

2. Add `prometheus Helm repo`

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

3. Create `Prometheus namespace`

```bash
kubectl create namespace prometheus
```

4. Install `kube-prometheus` stack

```bash
helm install stable prometheus-community/kube-prometheus-stack -n prometheus
```

5. Edit the service and make it `LoadBalancer`

```bash
kubectl edit svc stable-kube-prometheus-sta-prometheus -n prometheus

# In "vi":
# 1. Press "i" to enter insert mode.
# 2. Make your changes:

# replace:
# "type: ClusterIP"
# with:
# "type: LoadBalancer"

# 3. Press ESC to exit insert mode.
# 4. Type ":wq!" and press ENTER to save and exit.
```

6. Edit the Grafana service too to change it to `LoadBalancer`

```bash
kubectl edit svc stable-grafana -n prometheus
# In "vi":
# 1. Press "i" to enter insert mode.
# 2. Make your changes:

# replace:
# "type: ClusterIP"
# with:
# "type: LoadBalancer"

# 3. Press ESC to exit insert mode.
# 4. Type ":wq!" and press ENTER to save and exit.

# Test:
kubectl get svc -n prometheus
# *** output (example) ***
# NAME                                      TYPE           CLUSTER-IP       EXTERNAL-IP                                                                  PORT(S)                         AGE
# ...
# stable-grafana                            LoadBalancer   172.20.133.223   a45a09fd2f627430086d56295e0c7fcc-907585449.eu-central-1.elb.amazonaws.com    80:30834/TCP                    17m
# ...
# stable-kube-prometheus-sta-prometheus     LoadBalancer   172.20.165.54    a5dae8316e85944b3891c034b19be6ad-1063334104.eu-central-1.elb.amazonaws.com   9090:30209/TCP,8080:30517/TCP   17m
# ...
```

For `Grafana`:
Open: `http://a45a09fd2f627430086d56295e0c7fcc-907585449.eu-central-1.elb.amazonaws.com`
- username: `admin`
- pass: `prom-operator`
- Visit example dashboard: `Home > Dashboards > Kubernetes / Kubelet`
- Create new dashboard: `Home > Dashboards > New dashboard` -> `Import a dashboard`
- In `Find and import dashboards for common applications set`: `12740` 
   and press `LOAD` button. Select data source as `prometheus` and click
   `IMPORT`.

For `Prometheus`:
Open: `http://a5dae8316e85944b3891c034b19be6ad-1063334104.eu-central-1.elb.amazonaws.com:9090`

## Step 9: Deploy ArgoCD on EKS to fetch the manifest files to the cluster

1. Create a `namespace argocd`

On Ec2 run:

```bash
kubectl create namespace argocd
```

2. Add argocd repo locally

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.4.7/manifests/install.yaml

```

3. By default, `argocd-server` is not publically exposed. In this scenario, use a Load Balancer to make it usable:

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

4. Get the load balancer hostname using the command below:

```bash
kubectl get svc argocd-server -n argocd -o json
# *** oputput (example)
# ...
#    "status": {
#        "loadBalancer": {
#            "ingress": [
#                {
#                    "hostname": "a304cd3b3d7184132b011b6f4e486c5f-184089738.eu-central-1.elb.amazonaws.com"
# ...
```

**NOTE**: Wait a moment (this may take a few moments to finish)...

Open: `http://a304cd3b3d7184132b011b6f4e486c5f-184089738.eu-central-1.elb.amazonaws.com`

Once you get the load balancer hostname details, you can access the `ArgoCD dashboard` through it.

We need to enter the `Username` and `Password` for `ArgoCD`. The `username` will be `admin` by default. For the `password`, we need to run the command below:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
# ***output (example) ***
# bd2R4W84MqYoKPKF
```


5. Create new application - click: `NEW APP` and set:
- Application name: `react-app-example`
- Project name: `default`
- SYNC POLICY: `Automatic`
- Repository URL: `https://github.com/dariusz-trawicki/react-app-devsecops-deployment`
- Path: `configuration-files`
- Cluster URL: `https://kubernetes.default.svc`
- Namespaces: `default`

and click `CREATE` button.


## TESTS

1. In Grafana UI, open dashboards:

- `Home > Dashboards > Kubernetes / Compute Resources / Pod`
- `Home > Dashboards > Kubernetes / Networking / Namespace (Pods)`

2. Get the `public DNS` of Elastic Load Balancer

In `EC2 CLI` run:

```bash
kubectl get svc
# NAME                TYPE           CLUSTER-IP      EXTERNAL-IP                                                                  PORT(S)          AGE
# kubernetes          ClusterIP      172.20.0.1      <none>                                                                       443/TCP          131m
# react-app-example   LoadBalancer   172.20.28.149   a02a6ccf4196e4056b2b2770a9801306-1311311234.eu-central-1.elb.amazonaws.com   3000:31822/TCP   9m39s
```

**NOTE**: Wait a moment (this may take a few moments to finish)...

3. Open the application in your browser:
`http://a02a6ccf4196e4056b2b2770a9801306-1311311234.eu-central-1.elb.amazonaws.com:3000`

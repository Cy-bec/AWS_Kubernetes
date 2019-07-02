# Creating new clusters, configure structures, install tools and deploy own projects

## Table of Contents

- [Install necessary tools and establish connection](#install-necessary-tools-and-establish-connection)
- [Create your Amazon EKS Cluster and Worker Nodes](#create-your-amazon-eks-cluster-and-worker-nodes)
- [Test if your cluster is working](#test-if-your-cluster-is-working)
  - [One-liner-cluster-test](#one-liner-cluster-test)
  - [Step by Step-cluster-test](#step-by-step-cluster-test)
  - [Clean up](#clean-up)
- [Deploy the Kubernetes Web UI (Dashboard)](#deploy-the-kubernetes-web-ui-dashboard)
  - [One-liner-dashboard](#one-liner-dashboard)
  - [Step by Step-dashboard](#step-by-step-dashboard)
- [Using Helm with Amazon EKS](#using-helm-with-amazon-eks)
  - [Sources](#sources)
  - [Installation](#installation)
    - [Linux](#linux)
    - [Mac](#mac)
    - [Helm plugin for local tiller](#Helm-plugin-for-local-tiller)
- [Kubernetes Metrics Server](#kubernetes-metrics-server)
  - [Install metrics-server](#install-metrics-server)
  - [Control Plane Metrics with Prometheus](#control-plane-metrics-with-prometheus)
- [Deploy own Docker](#deploy-own-docker)
  - [Create Repository in AWS-ECR](#Create-Repository-in-AWS-ECR)
- [Deployments Services and Pods (incomplete)](#Deployments-Services-and-Pods-incomplete)
  - [References](#References)
  - [Deployment](#Deployment)
- [Cleaning Up your Amazon ECS Resources](#Cleaning-Up-your-Amazon-ECS-Resources)

## Install necessary tools and establish connection

([docs](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html))

1. Create an account to access AWS (Amazon Web Services).
2. Create the required client security token for cluster API server communication:
   1. ```pip3 install awscli --upgrade --user```
   2. ```aws --version```
   3. If you are unable to install version 1.16.156 or greater of the AWS CLI on your system, you must ensure that the AWS IAM Authenticator for Kubernetes is installed on your system. For more information, see [Installing aws-iam-authenticator](https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html).
   4. Configure Your AWS CLI Credentials in your environment
      1. `aws configure`
         1. AWS Access Key ID [None]: *__Put your account key ID here__*
         2. AWS Secret Access Key [None]: *__Put your secret access key here__*
         3. Default region name [None]: *__Put the region where you want to deploy it here__*
            1. [get region here](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.RegionsAndAvailabilityZones.html)
            2. e.g. Tokyo is `ap-northeast-1`
         4. Default output format [None]: *__json__*
   5. Install the `eksctl` command line utility
      1. `curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp`
      2. `sudo mv /tmp/eksctl /usr/local/bin`
      3. `eksctl version`
   6. Install and Configure kubectl for Amazon EKS (only Linux. See [kubernetes-docs](https://kubernetes.io/docs/tasks/tools/install-kubectl/) for more)
      1. latest: `curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl`
      2. Make the kubectl binary executable: `chmod +x ./kubectl`
      3. Move the binary in to your PATH: `sudo mv ./kubectl /usr/local/bin/kubectl`
      4. `kubectl version`

## Create your Amazon EKS Cluster and Worker Nodes

([git-docs](https://eksctl.io/))

```Console
eksctl create cluster \
--name prod \
--version 1.12 \
--nodegroup-name standard-workers \
--node-type t3.medium \
--nodes 3 \
--nodes-min 1 \
--nodes-max 4 \
--node-ami auto
```

 flag | description|
  --- | --- |
 --name string                    | EKS cluster name (generated if unspecified, e.g. "unique-creature-1561094398")
 --version string                 | Kubernetes version (valid options: 1.10, 1.11, 1.12) (default "1.12")
 --nodegroup-name string          | name of the nodegroup (generated if unspecified, e.g. "ng-80a14634")
 --node-type string               | node instance type (default "m5.large") [Amazon-docs](https://aws.amazon.com/ec2/pricing/on-demand/)
 --nodes int                      | total number of nodes (for a static ASG) (default 2) [Amazon-docs](https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html)
 --nodes-min int                  | minimum nodes in ASG (default 2)
 --nodes-max int                  | maximum nodes in ASG (default 2)
 --node-ami string                | Advanced use cases only. If 'static' is supplied (default) then eksctl will use static AMIs; if 'auto' is supplied then eksctl will automatically set the AMI based on version/region/instance type; if any other value is supplied it will override the AMI to use for the nodes. Use with extreme care. (default "static") ([Amazon-docs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html))

If the following Error appears follow [this](https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html) instructions to install it:

```Console
neither aws-iam-authenticator nor heptio-authenticator-aws are installed
```

Testing:

$ `kubectl get svc`

Example output:

```Console
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
kubernetes   ClusterIP   10.100.0.1      <none>        443/TCP    21h
```

## Test if your cluster is working

([Amazon-guide](https://docs.aws.amazon.com/eks/latest/userguide/eks-guestbook.html))

### One-liner-cluster-test

```Console
kubectl apply -f https://raw.githubusercontent.com/kubernetes/examples/master/guestbook-go/redis-master-controller.json && kubectl apply -f https://raw.githubusercontent.com/kubernetes/examples/master/guestbook-go/redis-master-service.json && kubectl apply -f https://raw.githubusercontent.com/kubernetes/examples/master/guestbook-go/redis-slave-controller.json && kubectl rolling-update redis-slave --image=k8s.gcr.io/redis-slave:v2 --image-pull-policy=Always && kubectl apply -f https://raw.githubusercontent.com/kubernetes/examples/master/guestbook-go/redis-slave-service.json && kubectl apply -f https://raw.githubusercontent.com/kubernetes/examples/master/guestbook-go/guestbook-controller.json && kubectl apply -f https://raw.githubusercontent.com/kubernetes/examples/master/guestbook-go/guestbook-service.json && kubectl get services -o wide --watch
```

### Step by Step-cluster-test

<details><summary>CLICK ME</summary><p>

1. Create the Redis master replication controller
   1. `kubectl apply -f https://raw.githubusercontent.com/kubernetes/examples/master/guestbook-go/redis-master-controller.json`
2. Create the Redis master service
   1. `kubectl apply -f https://raw.githubusercontent.com/kubernetes/examples/master/guestbook-go/redis-master-service.json`
3. Create the Redis slave replication controller
   1. `kubectl apply -f https://raw.githubusercontent.com/kubernetes/examples/master/guestbook-go/redis-slave-controller.json`
4. Update the container image for the Redis slave replication controller ([see](https://github.com/kubernetes/examples/issues/321))
   1. `kubectl rolling-update redis-slave --image=k8s.gcr.io/redis-slave:v2 --image-pull-policy=Always`
5. Create the Redis slave service
   1. `kubectl apply -f https://raw.githubusercontent.com/kubernetes/examples/master/guestbook-go/redis-slave-service.json`
6. Create the guestbook replication controller
   1. `kubectl apply -f https://raw.githubusercontent.com/kubernetes/examples/master/guestbook-go/guestbook-controller.json`
7. Create the guestbook service
   1. `kubectl apply -f https://raw.githubusercontent.com/kubernetes/examples/master/guestbook-go/guestbook-service.json`
8. Query the services in your cluster and wait until the External IP column for the guestbook service is populated
   1. `kubectl get services -o wide --watch`
9. After your external IP address is available, point a web browser to that address at port 3000 to view your guest book. For example, <http://a7a95c2b9e69711e7b1a3022fdcfdf2e-1985673473.us-west-2.elb.amazonaws.com:3000>

</p>
</details>

### Clean up

```Console
kubectl delete rc/redis-master rc/redis-slave rc/guestbook svc/redis-master svc/redis-slave svc/guestbook
```

## Deploy the Kubernetes Web UI (Dashboard)

([Amazon-guid](https://docs.aws.amazon.com/eks/latest/userguide/dashboard-tutorial.html))

### One-liner-dashboard

```Console
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml && kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/heapster.yaml && kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/influxdb.yaml && kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/rbac/heapster-rbac.yaml
```

Create a file called eks-admin-service-account.yaml with the text below. This manifest defines a service account and cluster role binding called eks-admin.

```Console
echo $'apiVersion: v1\nkind: ServiceAccount\nmetadata:\n  name: eks-admin\n  namespace: kube-system\n---\napiVersion: rbac.authorization.k8s.io/v1beta1\nkind: ClusterRoleBinding\nmetadata:\n  name: eks-admin\nroleRef:\n  apiGroup: rbac.authorization.k8s.io\n  kind: ClusterRole\n  name: cluster-admin\nsubjects:\n- kind: ServiceAccount\n  name: eks-admin\n  namespace: kube-system' > eks-admin-service-account.yaml
```

Apply the service account and cluster role binding to your cluster:

```Console
kubectl apply -f eks-admin-service-account.yaml
```

Copy token here:

```Console
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep eks-admin | awk '{print $1}')
```

Start server:

```Console
kubectl proxy
```

[Open Dashboard in default browser](http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/login)

### Step by Step-dashboard

<https://docs.aws.amazon.com/eks/latest/userguide/dashboard-tutorial.html>

## Using Helm with Amazon EKS

Use helm and tiller only on local machine!

### Sources

<https://medium.com/faun/helm-basics-using-tillerless-dac28508151f>

github: <https://github.com/rimusz/helm-tiller>

### Installation

#### Linux

```Console
sudo snap install helm --classic
```

#### Mac

```Console
brew install kubernetes-helm
```

#### Helm plugin for local tiller

Install:

```Console
helm plugin install https://github.com/rimusz/helm-tiller
```

Start local Tiller with the plugin (change *__local-tiller-namespace__* as you want):

```Console
helm tiller start local-tiller-namespace
```

Check release status:

```Console
helm status local-tiller-namespace
```

Stop local Tiller with the plugin:

```Console
helm tiller stop
```

Usage to use helm with plugin (replace *__HELM_COMMANDS__*):

```Console
helm tiller run my-tiller-namespace -- HELM_COMMANDS
```

Example:

```Console
helm tiller run my-tiller-namespace -- helm list
helm tiller run my-tiller-namespace -- bash -c 'echo running helm; helm list'
```

## Kubernetes Metrics Server

([Amazon-guide-metrics-server](https://docs.aws.amazon.com/eks/latest/userguide/metrics-server.html), [Amazon-guide-Prometheus](https://docs.aws.amazon.com/eks/latest/userguide/prometheus.html))

### Install metrics-server

Navigate to a directory where you would like to download the latest *metrics-server* release.

```Console
mkdir metrics-server && cd metrics-server
```

Make sure you have this tools:

1. `curl --version`
2. `tar --version`
3. `gzip --version`
4. `jq --version`

Download and apply *metrics-server*:

```Console
DOWNLOAD_URL=$(curl --silent "https://api.github.com/repos/kubernetes-incubator/metrics-server/releases/latest" | jq -r .tarball_url)
DOWNLOAD_VERSION=$(grep -o '[^/v]*$' <<< $DOWNLOAD_URL)
curl -Ls $DOWNLOAD_URL -o metrics-server-$DOWNLOAD_VERSION.tar.gz
mkdir metrics-server-$DOWNLOAD_VERSION
tar -xzf metrics-server-$DOWNLOAD_VERSION.tar.gz --directory metrics-server-$DOWNLOAD_VERSION --strip-components 1
kubectl apply -f metrics-server-$DOWNLOAD_VERSION/deploy/1.8+/
```

Verify that the metrics-server deployment is running the desired number of pods with the following command:

```Console
kubectl get deployment metrics-server -n kube-system
```

### Control Plane Metrics with Prometheus

 *__!!!Start helm with tiller plugin!!!__*

 Install Prometheus with helm plugin:

```Console
kubectl create namespace prometheus

helm tiller run my-tiller-namespace -- helm install stable/prometheus \
--name prometheus \
--namespace prometheus \
--set alertmanager.persistentVolume.storageClass="gp2",server.persistentVolume.storageClass="gp2"
```

Verify that all of the pods in the prometheus namespace are in the READY state:

```Console
kubectl get pods -n prometheus
```

Use kubectl to port forward the Prometheus console to your local machine:

```Console
kubectl --namespace=prometheus port-forward deploy/prometheus-server 9090
```

Open Prometheus console in default browser: [localhost:9090](http://localhost:9090)

## Deploy own Docker

### Create Repository in AWS-ECR

[Amazon-guide](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/docker-basics.html#use-ecr)

Create repository. (replace *__hello-repository__* and *__region__*)

```Console
aws ecr create-repository --repository-name hello-repository --region region
```

|flag| description |
|--- | ---|
|--repository-name (string) | The name to use for the repository. The repository name may be specified on its own (such as nginx-web-app ) or  it  can  be  prepended with  a  namespace  to group the repository into a category (such as project-a/nginx-web-app )|
|--region | the region where it will be created.  e.g. Tokyo is `ap-northeast-1`. [get region here](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.RegionsAndAvailabilityZones.html)|

Output (Note the repositoryUri in the output. It will be used to push Dockers):

```Console
{
    "repository": {
        "registryId": "aws_account_id",
        "repositoryName": "hello-repository",
        "repositoryArn": "arn:aws:ecr:region:aws_account_id:repository/hello-repository",
        "createdAt": 1505337806.0,
        "repositoryUri": "aws_account_id.dkr.ecr.region.amazonaws.com/hello-repository"
    }
}
```

Tag docker-image with with the repositoryUri value:

Example:

docker-image = *__hello-world__*

repositoryUri = *__aws_account_id.dkr.ecr.region.amazonaws.com/hello-repository__*

```Console
docker tag hello-world aws_account_id.dkr.ecr.region.amazonaws.com/hello-repository
```

Run the aws ecr get-login --no-include-email command to get the docker login authentication command string for your registry

Change *__region__* to the region of the docker-repository

```Console
aws ecr get-login --no-include-email --region region
```

Run the docker login command that was returned in the previous step. This command provides an authorization token that is valid for 12 hours.

Push the image to Amazon ECR with the repositoryUri value from the earlier step

Example:
Change *__aws_account_id.dkr.ecr.region.amazonaws.com/hello-repository__* to your docker-repository

```Console
docker push aws_account_id.dkr.ecr.region.amazonaws.com/hello-repository
```

## Deployments, Services and Pods (incomplete)

### References

<https://www.bogotobogo.com/DevOps/Docker/Docker_Kubernetes_NodePort_vs_LoadBalancer_vs_Ingress.php>

<https://kubernetes.io/docs/concepts/services-networking/connect-applications-service/>

<https://kubernetes.io/docs/concepts/services-networking/service/>

<https://kubernetes.io/docs/concepts/cluster-administration/networking/>

<https://kubernetes.io/docs/tasks/access-application-cluster/communicate-containers-same-pod-shared-volume/>

<https://kubernetes.io/docs/tasks/access-application-cluster/connecting-frontend-backend/>

<https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/>

### Deployment

A deployment is basically the receipt and the manager for the pods.
It will create the connections, replica sets, the ports, the amount of pods, etc. ...

Example (will create 5 pods from my docker-image on ecr):

```Yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  selector:
    matchLabels:
      app: hello
      tier: frontend
      track: stable
  replicas: 5
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: hello
        tier: frontend
        track: stable
    spec:
      containers:
        - name: hello
          image: "COPY_HERE_DOCKER_IMAGE_URL_FROM_ECR"
          ports:
            - containerPort: 5000
          resources:
            limits:
               memory: "128Mi"
               cpu: "400m"
          livenessProbe:
            httpGet:
               path: /
               port: 5000
            initialDelaySeconds: 30
            periodSeconds: 30
          readinessProbe:
            httpGet:
               path: /
               port: 5000
            initialDelaySeconds: 15
            periodSeconds: 3
```

Create service to access from other pods (to make it accessible from outer-world add `type: LoadBalancer` in spec: ):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-frontend
  labels:
    run: service-frontend
spec:
  selector:
    app: hello
    tier: frontend
  ports:
  - protocol: "TCP"
    port: 5000
    targetPort: 5000
```

Create ssh pod to test the connection:

```Console
kubectl apply -f https://k8s.io/examples/application/shell-demo.yaml

kubectl get pod shell-demo --watch

kubectl exec -it shell-demo -- /bin/bash
```

in the shell:

```Console
apt update

apt install curl --yes

printenv | grep SERVICE

curl -v $host_adresss:$and_hostPort
```

## Cleaning Up your Amazon ECS Resources

([Amazon-guide](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CleaningUp.html))

# Deploy your Docker container on Kubernetes in AWS using KOPS (Kubernetes Operations)

NOTE: *Several dollars per day* will easily be charged by AWS for the components that are provisioned once you run the following

`kops update cluster --name ${KOPS_CLUSTER_NAME} --yes`

To tear the cluster down at any time, once it is running, run the following command when you wish to stop the billing, of course after replacing that variable with your own

`kops delete cluster --name ${KOPS_CLUSTER_NAME} --yes`

---
## The steps
- [x] Have a docker image to serve out http / json so that you can look at your successfully K8s-deployed content
- [x] Have an account at cloud computing provider, this text assumes AWS
- [ ] Install, configure AWS CLI
- [ ] Provision an AWS S3 bucket to hold K8s's state and version it
- [ ] Create ssh keys for KOPS
- [ ] Install KOPS
- [ ] *Define* the Kubernetes configuration (`kops create...`)
- [ ] Use that definition to *create* the cluster (`kops update...`)
- [ ] Install Helm
- [ ] Use Helm to install an ingress-controller
- [ ] Deploy a Kubernetes sevice out of your Docker image 
- [ ] Deploy an ingress controller that forwards its traffic to your service

(Every detail is best assumed as subject to change by all involved parties and vendors.)

---

### Install AWS CLI ([link](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html))

### Configure AWS CLI ([link](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html))

---

### Provision an S3 to hold the K8s state

Define some variables to avoid repetition later, here filled with example values
```
export KOPS_STATE_S3_NAME=hold-myk8s-state
export KOPS_STATE_STORE=s3://hold-myk8s-state
export KOPS_CLUSTER_NAME=myk8s.k8s.local
export KOPS_SSH_FILE_PATH=~/.ssh/kops_rsa
export KOPS_DESIRED_REPLICA_COUNT=2
```

then

```
aws s3api create-bucket \
    --bucket ${KOPS_STATE_S3_NAME} \
    --region <your region> \
    --create-bucket-configuration LocationConstraint=<your region>

aws s3api put-bucket-versioning \
    --bucket ${KOPS_STATE_S3_NAME} \
    --versioning-configuration Status=Enabled
```

---

### Create SSH keys for KOPS ([link](https://www.ssh.com/ssh/keygen/))

---

### Install KOPS ([link](https://kops.sigs.k8s.io/getting_started/install/))

---

### KOPS up your K8s cluster ([link](https://kops.sigs.k8s.io/getting_started/aws/))

Here with example params and values, see `kops create cluster --help` for more
```
kops create cluster \
    --cloud=aws \
    --name=${KOPS_CLUSTER_NAME} \
    --ssh-public-key=${KOPS_SSH_FILE_PATH} \
    --zones=<one or many zones in your region> \
    --state=${KOPS_STATE_STORE} \
    --master-size=t2.micro \
    --node-count=${KOPS_DESIRED_REPLICA_COUNT} \
    --node-size=t2.micro

kops update cluster --name ${KOPS_CLUSTER_NAME} --yes
```

Wait for about 10 mins. Poking the process too early or even accidentally issuing further commands to the cluster prematurely might actually hurt or corrupt the automated process. Meanwhile it might be exciting to go to your AWS Console to observe all the resources that are now being provisioned to us by KOPS.

Verify the cluster is up and running
```
kops validate cluster --name ${KOPS_CLUSTER_NAME}
```

Once it is, see the components details
```
clear && kubectl get all && kubectl get ing
```

Once validation shows that everything is running ok you may start `kubectl` -ing to yours heart's content.

This command may be heplful in keeping track of the commands that you have run

`history -w /dev/stdout | sort | uniq | grep ^kubectl`

---

### Install Helm ([link](https://helm.sh/docs/intro/install/))

---

### Install nginx ingress-controller using Helm ([link](https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-helm/))

---

### Deploy a sevice out of your Docker image 

See associatd sample deployment yaml file.
```
kubectl apply -f my-deployment.yaml
```

This command will create the deployment, replicaset, services, and the pods, which will then `pull` the Docker images and run them as containers. Be mindful of all those **ports** all the way up from what your service listens to, what your docker image exposes, what you set for your service here, which port you forward traffic from ingress-controller, and what that controller listens to.

---

### Deploy ingress controller

See how the associated ingress-controller yaml file defines the rule to forward its traffic to the deployment's service.

Set the ingress-controller's spec.rules.host to the DNS address of the
- AWS Load Balancer
- of your KOPS K8s deployment
- that is internet facing
- and serves to the nodes
- of this KOPS K8s instance.

```
kubectl apply -f my-ingress-controller.yaml
```


Monitor the progress

```
clear && kubectl get all && kubectl get ing
```

If something went wrong, you could first try to right them up at those deployment and ingress yamls and `kubectl apply -f...` them again. Once everything is running and you have set all the keys, Ports, and DNS correctly...

---

### Navigate to your service

Navigate to the DNS of that ingress controller LB (eg. with broswer or Postman), remember to add protocol to the url, and see your service responding to you your content in verification that your K8s deployment including its ingress and your Docker container therein are all working.

---

### Finally teardown your cluster

Once you're ready, tear down your cluster to stop the billing for any priced cloud components and observe those components spin down in the AWS Console.

```
kops delete cluster --name ${KOPS_CLUSTER_NAME} --yes
```

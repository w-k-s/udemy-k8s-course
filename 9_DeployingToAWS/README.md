# 9. Deploying to AWS

- K8s works with a number of cloud platforms e.g. AWS, Azure, Google Cloud.
- On AWS, when using k8s, we will not need to worry about creating EC2 instances, load balancers or autoscale groups. k8s will manage all of this. In other words, k8s abstracts away the intrinsics of the cloud platform.

## 9.1 Nodes & EBS

**No more minikube**

- Up until now, we've been using Minikube running inside a Virtual Machine. 
- Minikube contained our worloads, and some of the directories on the pod were mapped to the host filesystem.
- Minikube however is only for local development. For production system, we use **Nodes**.

**Node**

- A node is a physical server in our system. In the case of AWS, a node corresponds to an EC2 instance.
- Typically, we would have multiple nodes in a cluster, each node containing one or two services
- The advantage of using multiple nodes are each node can be a micro instance.
- These nodes will be managed by a master node that is respondible for scheduling. Meaning **the master node will decide which node a particular pod will run on**.
- The master node will also handle node failure. For instance, if the API Gateway node fails, it will respawn the node on a different running node. Alternatively, if we are using replicasets to avoid zero down-time, the master node will allocate the pod on two different nodes.

**EBS**

- In the previous chapter, we mapped pod storage to the host filesystem storage.
- Now we will map the pod storage to Elastic Block Store.

## 9.2 Introducing kops

- Installing kubernetes on AWS is no easy task, if one were to try to do it manually.
- `kops` is the easiest way to get a production grade Kubernetes cluster up and running. It can be thought of as kubectl for clusters.

### 9.2.1 Installing kops

- Install `kops` on an AWS AMI EC2 instance. This would ideally be a small instance e.g. `t2-nano`.
- At this point in time, we do not need to configure instance details or add storage. It's a good idea to tag the instance to keep track of where the charges are coming from and to configure a security group (only SSH is required).
- Once the EC2 instance is running, ssh into the instance and follow the instructions to install kops on linux ([available here](https://github.com/kubernetes/kops))

```
curl -LO https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64
chmod +x kops-linux-amd64
sudo mv kops-linux-amd64 /usr/local/bin/kops
```

### 9.2.2 Installing kubectl

- Once `kops` has been installed, we'll also need to install `kubectl`. Instructions to install kops on linux are available [here](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html).

```
curl -o kubectl.sha256 https://amazon-eks.s3-us-west-2.amazonaws.com/1.11.5/2018-12-06/bin/linux/amd64/kubectl.sha256
chmod +x ./kubectl
mkdir $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$HOME/bin:$PATH
echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc
kubectl version --short --client
```

### 9.2.3 Granting `kops` permissions

To continue to set up `kops` on AWS, follow the setup instructions available [here](https://github.com/kubernetes/kops/blob/master/docs/aws.md):

- `kops` needs a set of permissions in order to function:

```
aws iam create-group --group-name kops

aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/IAMFullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess --group-name kops

aws iam create-user --user-name kops

aws iam add-user-to-group --user-name kops --group-name kops

aws iam create-access-key --user-name kops
```

- You should record the SecretAccessKey and AccessKeyID in the returned JSON output, and then use them below:

```
# configure the aws client to use your new IAM user
aws configure           # Use your new access and secret key here
aws iam list-users      # you should see a list of all your IAM users here

# Because "aws configure" doesn't export these vars for kops to use, we export them now
export AWS_ACCESS_KEY_ID=$(aws configure get aws_access_key_id)
export AWS_SECRET_ACCESS_KEY=$(aws configure get aws_secret_access_key)
```
### 9.2.4 Configuring DNS

 If you are using Kops 1.6.2 or later, then DNS configuration is optional. Instead, a gossip-based cluster can be easily created. 

 The only requirement to trigger this is to have the cluster name end with .k8s.local. If a gossip-based cluster is created then you can skip this section


### 9.2.5 Cluster state storage

- We need to configure an S3 bucket where `kops` can store the state of a cluster.

```
aws s3api create-bucket \
    --bucket prefix-example-com-state-store \
    --region us-east-1
```

Note: S3 requires `--create-bucket-configuration LocationConstraint=<region>` for regions other than us-east-1.

```
aws s3api create-bucket --bucket prefix-example-com-state-store --create-bucket-configuration LocationConstraint=ap-south-1
```

### 9.2.6 Prepare Local Environment

This is the last step before we can create a cluster.

```
export NAME=myfirstcluster.k8s.local
export KOPS_STATE_STORE=s3://prefix-example-com-state-store
```

Export the name of your cluster, which must end with `k8s.local` if you skipped DNS configuration.

Also export the path to s3 bucket created for kops to store cluster state.

### 9.2.7 Create Cluster Configuration

Kops is able to create cluster configurations. One of the parameters this expects is the availability zones to use for your region. We can find out the availability zones using the following command:

```
aws ec2 describe-availability-zones --region ap-south-1
```

Example
```
{
    "AvailabilityZones": [
        {
            "State": "available", 
            "ZoneName": "ap-south-1a", 
            "Messages": [], 
            "RegionName": "ap-south-1"
        }, 
        {
            "State": "available", 
            "ZoneName": "ap-south-1b", 
            "Messages": [], 
            "RegionName": "ap-south-1"
        }
    ]
}
```

We can then ask kops to create configuration files using all available availability zones:
```
kops create cluster --zones ap-south-1a,ap-south-1b ${NAME}
```

This may cause an error:

```
SSH public key must be specified when running with AWS (create with `kops create secret --name mycluster.k8s.local sshpublickey admin -i ~/.ssh/id_rsa.pub`)
```

An ssh key must be created for the cluster. This can be done using the following command:

```
ssh-keygen -b 2048 -t rsa -f ~/.ssh/id_rsa
```

Then run the command mentioned in the error message:
```
kops create secret --name ${NAME} sshpublickey admin -i ~/.ssh/id_rsa.pub
```

To verify or edit the cluster configuration files, enter

```
kops edit cluster ${NAME}
```

**IMPORTANT**. Kops creates an instance group for the master instance  and an instance group for all the nodes that it creates. To see the instance groups, enter the command:

```
kops get ig --name ${NAME}
```

Ouput: 

```
NAME			ROLE	MACHINETYPE	MIN	MAX	ZONES
master-ap-south-1a	Master	c4.large	1	1	ap-south-1a
nodes			Node	t2.medium	3	5	ap-south-1a,ap-south-1b
```

We can edit each of these instance groups using the command:

```
kops edit ig <IG_NAME> --name ${NAME}
```
Example:

```
$ kops edit ig nodes --name ${NAME}
apiVersion: kops/v1alpha2
kind: InstanceGroup
metadata:
  creationTimestamp: 2019-02-02T13:43:59Z
  labels:
    kops.k8s.io/cluster: mycluster.k8s.local
  name: nodes
spec:
  image: kope.io/k8s-1.11-debian-stretch-amd64-hvm-ebs-2018-08-17
  machineType: t2.medium
  maxSize: 2
  minSize: 2
  nodeLabels:
    kops.k8s.io/instancegroup: nodes
  role: Node
  subnets:
  - ap-south-1a
  - ap-south-1b
```

- The machineType specifies what instances will be created for each one of our nodes. We could change this to a bigger or smaller instance.

- maxSize/minSize is the maximum /mininum number of nodes in the instance group. We will probably need many nddes for this tutorial so change these values to 5 and 3 respectively.

We do not need to edit the master instance group for now, but we can still view the configuration using the same command mentioned earlier passing in the name of the `master` instance group instead of the `nodes` instance group.

```
$ kops edit ig master-ap-south-1a --name ${NAME}
apiVersion: kops/v1alpha2
kind: InstanceGroup
metadata:
  creationTimestamp: 2019-02-02T13:43:59Z
  labels:
    kops.k8s.io/cluster: mycluster.k8s.local
  name: master-ap-south-1a
spec:
  image: kope.io/k8s-1.11-debian-stretch-amd64-hvm-ebs-2018-08-17
  machineType: c4.large
  maxSize: 1
  minSize: 1
  nodeLabels:
    kops.k8s.io/instancegroup: master-ap-south-1a
  role: Master
  subnets:
  - ap-south-1a
```

A `c4.large` instance will be created to manage the cluster.

### 9.2 Running the cluster

### 9.3 Deleting the cluster

### 9.4 Restarting the cluster
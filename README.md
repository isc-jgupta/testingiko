# Configuring SCO Topology using IKO 

## Steps to configure SCO EKS Cluster

1. Setting up EKS Cluster (Can be modified to work on other cloud services)
2. Installing IKO
3. Modifying IrisCluster and IrisService

## Prerequisites

1. The following libraries in your terminal:
       eksctl
       kubectl
       AWS CLI
       helm
2. ICR (InterSystems Container Registry) access
3. An IRIS key

## 1. Setting up EKS Cluster

Make sure to edit the eks-cluster.yaml according to the comments. Once done, you can run the following command

```
eksctl create cluster -f eks-cluster.yaml
```

The cluster by default creates a [gp2 storage class](https://aws.amazon.com/ebs/volume-types/#:~:text=gp2%20is%20the%20default%20EBS,interactive%20applications%2C%20and%20boot%20volumes) which should be sufficient for most operations. 

You can check the status of the cluster creation by looking at the status in the AWS management portal.

## 2. Installing IKO

We'll need to download the IRIS container from ICR, which means we need credentials installed in Kubernetes. The IKO pods will be created within the IKO namespace.

```
kubectl create namespace iko
kubectl create secret docker-registry dockerhub-secret -n iko --docker-server=https://containers.intersystems.com/v2/ --docker-username <YOUR USERNAME> --docker-password='<YOUR PASSWORD>'
helm install intersystems chart/iris-operator -n iko
```

## 3. Modifying IrisCluster and IrisService

The cluster is being deployed in the iko namespace. This can easily be changed however by making sure the docker secret and the configurations in this section are in the namespace you want it to be (determined by the parameter fed into the -n flag).

### Deploying IRIS Key

IRIS requires a valid license key, so let's take the license key and add it to Kubernetes so we can use it in our IRIS cluster.  Note:  The file *must* be named `iris.key`.

```
kubectl create secret generic iris-key-secret -n iko --from-file=iris.key
```

### Creating a configmap with your IRIS CPF and CSP-merge.ini files

InterSystems uses CPF programatically configure IRIS. The `common.cpf` file contains CPF parameters that are applied to all IRIS instances in the _IrisCluster_.  

1. Create a password hash for your IRIS by running the following
```
echo "MySecretPassword" | docker run -i containers.intersystems.com/intersystems/passwordhash:1.1
```

2. Edit the included example `common.cpf` file to include the password hash you just created

3. The `CSP-merge.ini` file contains changes to be applied to the web gateway configuration. The example file sets the password for the different LoadBalancer services that deploy in this script. Note that the file needs to changed to accomodate for the different name of the pods (if you change the parameter _metadata.name_ in the IrisCluster.yaml file)

Create the configmap
```
kubectl create cm -n iko sco-cpf --from-file common.cpf --from-file CSP-merge.ini
```

### Installing the cluster 

The IRIS Kubernetes Operator can deploy enterprise IRIS installations via a series of kubernetes resources.

Let's create a cluster
```
kubectl apply -f IrisCluster.yaml
kubectl apply -f IrisService.yaml
```

# Issues that were resolved and have yet to be resolved

## Resolved

Problem: Persistent Volumes were not being created in the AWS Portal <br />
Solution: Using the EKS Cluster definition allowed for the creation of PVs dynamically<br />

Problem: Data Pods were not being mirrored (having mirror param config issues during container setup within the pods<br />
Solution: Using IRIS Version 2023.1.1.380.0 for the iris containers as well as all the other IRIS products<br />

Problem: Management Portal for compute node was not appearing <br />
Solution: Changing the password of the Management Portal through the webgateway that is accessible<br />

## Unresolved

Here are a list of problems that still needs to be resolved:

1. Mirroring IRIS pods running the latest version of IRIS <br />
2. Understanding why the SC namespace is not showing up on the data pods <br />
3. Making sure the SC namespace is showing up in the mirror <br />
4. Testing the ECP 



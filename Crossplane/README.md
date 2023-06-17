Crossplane:

Crossplane links your Kubernetes cluster to non-kubernetes external resources and enables platform teams to create custom Kubernetes APIs to consume those resources.

Crossplane allows you to provide, compose, and consume the infrastructure of any cloud service provider using the Kuberntes API.

Crossplane also acts as a Kubernetes Controller to watch the state of the external resources and provide state enforcement. If something changes or removes a resource other than Kubernetes, Crossplane reverses the change or recreates the deleted resource.

What’s that whole story about?
Install Cross plane on our Kubernetes Cluster ( AKS, GKE, EKS, KIND )
Configure Crossplane to communicate with AWS.
Install required packages for Crossplane to communicate with AWS.
Create a S3 bucket using Crossplane from our Kubernetes Cluster.
Verify that the resources have been created from the AWS Console.
Prerequisites
A Kubernetes cluster with at least 6 GB of RAM ( Can be either On-Prem, AKS, EKS, GKE, Kind )
Permissions to create pods and secrets in the Kubernetes cluster
Helm version v3.2.0 or later
An AWS account with permissions to resources
AWS access keys
Install Crossplane
Crossplane installs into an existing Kubernetes cluster.

Once your kubernetes setup is ready let’s install Crossplane in cluster.

#Create the namespace and install the components using helm

kubectl create namespace crossplane-system

helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

helm install crossplane --namespace crossplane-system crossplane-stable/crossplane
Verify Crossplane installed with kubectl get pods. Output should like below.


Install the AWS provider
Now get into install the AWS provider into the kubernetes cluster using the below yaml file.

apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  package: xpkg.upbound.io/upbound/provider-aws:v0.27.0
Verify the provider using the kubectl get proivders command.


Create a Kubernetes secret for AWS
For the creation and administration of AWS resources, the supplier needs credentials. Credentials are connected to providers via a Kubernetes Secret.

From your AWS key-pair, first create a Kubernetes Secret, and then configure the Provider to use it.

Generate an AWS key-pair file
Create a text file containing the AWS account aws_access_key_id and aws_secret_access_key.

[default]
aws_access_key_id = <aws_access_key>
aws_secret_access_key = <aws_secret_key>
Save this text file as aws-credentials.txt

Create a Kubernetes secret with the AWS credentials
A Kubernetes generic secret has a name and contents. Use kubectl create secret to generate the secret object named aws-secret in the crossplane-system namespace. Use the --from-file= argument to set the value to the contents of the aws-credentials.txt file.

kubectl create secret generic aws-secret -n crossplane-system --from-file=creds=./aws-credentials.txt
Verify the created secret using the below commands.

kubectl get secret -n crossplane-system

root@kube-master:~/crossplane# kubectl get secret -n crossplane-system
NAME                               TYPE                 DATA   AGE
aws-secret                         Opaque               1      25s
kubectl describe secret aws-secret -n crossplane-system

root@kube-master:~/crossplane# kubectl describe secret aws-secret -n crossplane-system
Name:         aws-secret
Namespace:    crossplane-system
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
creds:  116 bytes
Create a ProviderConfig
A ProviderConfig customizes the settings of the AWS Provider.

Create ProviderConfig yaml file and apply with the command:

apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-secret
      key: creds
secretRef- This attach AWS credentials which is saved in kubernetes secret.

Create a managed resource in AWS:
A managed resource is anything Crossplane creates and manages outside of the Kubernetes cluster. This creates an AWS S3 bucket with Crossplane. The S3 bucket is a managed resource.

apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  name: $bucket
spec:
  forProvider:
    region: us-east-2
  providerConfigRef:
    name: default
The apiVersion and kind are from the provider’s CRDs.

The metadata.name value is the name of the created S3 bucket in AWS.

So, once applied Use kubectl get buckets to verify Crossplane created the bucket.

root@kube-master:~/crossplane# kubectl get buckets
NAME         READY   SYNCED   EXTERNAL-NAME   AGE
selva-demo   True    True     selva-demo      103s
Note: Bucket status should be true in READY and SYNCED.

Let’s verify the bucket in AWS console.


Delete the managed resource:
Before you close your work delete the managed resources what you’ve created Using the below command.

root@kube-master:~/crossplane# kubectl delete bucket selva-demo
bucket.s3.aws.upbound.io "selva-demo" deleted

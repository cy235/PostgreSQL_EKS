# PostgreSQL_EKS
## Install and Configure OpenEBS on Amazon Elastic Kubernetes Service
```
eksctl create cluster \
 --name postgreEKS-demo \
 --version 1.14 \
 --nodegroup-name ng-workers \
 --node-type t3.medium \
 --nodes 3 \
 --nodes-min 3 \
 --nodes-max 6 \
 --node-ami auto \
 --node-ami-family Ubuntu1804 \
 --set-kubeconfig-context=true
```
```
kubectl get nodes
```
```
aws ec2 describe-instances \
        --query 'Reservations[].Instances[].[Tags[?Key==`Name`].Value[] | [0], InstanceId, Placement.AvailabilityZone]' \
        --output text
```
```
aws ec2 create-volume \
    --size 20 \
    --availability-zone us-east-1b
```
```
aws ec2 attach-volume \
	--volume-id vol-???? \
	--instance-id i-???? \
	--device /dev/sdf
```

## Deploy PostgreSQL on Kubernetes Running the OpenEBS Storage Engine

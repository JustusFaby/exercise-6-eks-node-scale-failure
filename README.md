# Exercise 6: EKS Node Scale Failure

## Objective
The objective of this exercise is to simulate, investigate, and resolve an Amazon EKS scaling failure where the Horizontal Pod Autoscaler (HPA) triggers a need for more resources, but the Cluster Autoscaler fails to provision new EC2 instances due to misconfigured node group settings (specifically, missing Auto Scaling Group tags).

## Architecture
- **EKS Cluster**: `exercise6-cluster` (Version 1.36) in `us-east-1`
- **Managed Node Group**: `workers` (Instance type: `t3.small`)
- **Cluster Autoscaler**: Deployed as a Deployment in `kube-system` using IRSA (IAM Roles for Service Accounts)
- **Application**: Nginx Deployment configured with HPA and a Load Generator to simulate traffic and CPU usage.

## Prerequisites
- An active EKS Cluster (`exercise6-cluster`) with an OIDC provider associated.
- Metrics Server installed.
- AWS CLI, `kubectl` configured.

## Steps to Reproduce the Incident

1. **Configure IAM for Cluster Autoscaler**:
   Created an IAM Policy and an IAM Role trusted by the EKS OIDC provider to allow the Cluster Autoscaler to modify ASGs.

2. **Induce the Incident (Misconfigure ASG Tags)**:
   Removed the `k8s.io/cluster-autoscaler/enabled` tag from the `eks-workers` Auto Scaling Group to prevent the CA from discovering it.

3. **Deploy Cluster Autoscaler**:
   Deployed the Cluster Autoscaler targeting the `exercise6-cluster` with auto-discovery based on tags.

4. **Deploy Application and HPA**:
   Created a namespace `exercise6`, deployed an `nginx` application requesting `500m` CPU per pod, and configured an HPA to scale out when CPU utilization reaches 50%.

5. **Generate CPU Load**:
   Ran a `load-generator` pod looping HTTP requests to the `nginx` service, causing the HPA to increase the desired replica count.

## Investigation
- Ran `kubectl get hpa -n exercise6` and saw the desired replicas increase while current replicas remained low.
- Ran `kubectl get pods -n exercise6` and saw new pods stuck in the `Pending` state.
- Ran `kubectl describe pod <pending-pod> -n exercise6` and observed `FailedScheduling` events citing `0/1 nodes are available: 1 Insufficient cpu`.
- Checked Cluster Autoscaler logs: `kubectl logs deployment/cluster-autoscaler -n kube-system`. The logs did not show any `Scale-up` events and failed to find the target node group due to missing configuration tags.

## Root Cause
The Horizontal Pod Autoscaler correctly calculated that more pods were needed to handle the CPU load, but the cluster lacked sufficient compute resources (`t3.small` nodes). The **Cluster Autoscaler** could not provision new nodes because the Auto Scaling Group was missing the `k8s.io/cluster-autoscaler/enabled=true` tag. Therefore, the CA could not discover the node group to increase its desired capacity.

## Resolution
Added the `k8s.io/cluster-autoscaler/enabled=true` tag back to the EKS Managed Node Group's Auto Scaling Group.

## Commands Used

### 1. Inducing the Failure (Removing ASG Tag)
```bash
aws autoscaling delete-tags --tags "ResourceId=<ASG-ID>,ResourceType=auto-scaling-group,Key=k8s.io/cluster-autoscaler/enabled"
```

### 2. Generating Load
```bash
kubectl run load-generator --image=busybox:1.28 --restart=Never -n exercise6 -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://nginx; done"
```

### 3. Investigation
```bash
kubectl get hpa -n exercise6
kubectl get pods -n exercise6
kubectl describe pod <pending-pod> -n exercise6
kubectl logs deployment/cluster-autoscaler -n kube-system
```

### 4. Resolution
```bash
aws autoscaling create-or-update-tags --tags "ResourceId=<ASG-ID>,ResourceType=auto-scaling-group,Key=k8s.io/cluster-autoscaler/enabled,Value=true,PropagateAtLaunch=true"
```

### 5. Verification
```bash
kubectl get nodes
kubectl get pods -n exercise6
```

## Expected Output
After resolving the issue, the Cluster Autoscaler logs show a `Scale-up` event. Running `kubectl get nodes` will reveal a new EC2 instance joining the cluster and transitioning to `Ready`. Finally, `kubectl get pods -n exercise6` will show the previously `Pending` pods transitioning to `Running`.

## Demo Video Link
[https://drive.google.com/file/d/1AkdDaiyTFZkePx0ZrW4XEDogqnc_5dAm/view?usp=sharing]

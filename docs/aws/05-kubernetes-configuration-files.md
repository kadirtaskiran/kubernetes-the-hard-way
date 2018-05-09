# Generating Kubernetes Configuration Files for Authentication

In this lab you will generate [Kubernetes configuration files](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/), also known as kubeconfigs, which enable Kubernetes clients to locate and authenticate to the Kubernetes API Servers.

## Client Authentication Configs

In this section you will generate kubeconfig files for the `kubelet` and `kube-proxy` clients.

> The `scheduler` and `controller manager` access the Kubernetes API Server locally over an insecure API port which does not require authentication. The Kubernetes API Server's insecure port is only enabled for local access.

### Kubernetes Public IP Address

Each kubeconfig requires a Kubernetes API Server to connect to. To support high availability the IP address assigned to the external load balancer fronting the Kubernetes API Servers will be used.

Retrieve the `kubernetes` static IP address:

```
KUBERNETES_PUBLIC_ADDRESS=$(aws elb describe-load-balancers --load-balancer-name kubernetes | jq -r '.LoadBalancerDescriptions[].DNSName')
```

### The kubelet Kubernetes Configuration File

When generating kubeconfig files for Kubelets the client certificate matching the Kubelet's node name must be used. This will ensure Kubelets are properly authorized by the Kubernetes [Node Authorizer](https://kubernetes.io/docs/admin/authorization/node/).

Generate a kubeconfig file for each worker node:

```
for instance in worker-1; do

  PRIVATE_HOSTNAME=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}"  \
    --query 'Reservations[].Instances[].[Tags[?Key==`PrivateHostName`].Value]' --output text)

  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=${PRIVATE_HOSTNAME}.kubeconfig

  kubectl config set-credentials system:node:${PRIVATE_HOSTNAME} \
    --client-certificate=${PRIVATE_HOSTNAME}.pem \
    --client-key=${PRIVATE_HOSTNAME}-key.pem \
    --embed-certs=true \
    --kubeconfig=${PRIVATE_HOSTNAME}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${PRIVATE_HOSTNAME} \
    --kubeconfig=${PRIVATE_HOSTNAME}.kubeconfig

  kubectl config use-context default --kubeconfig=${PRIVATE_HOSTNAME}.kubeconfig
done
```

Results:

```
worker-0.kubeconfig
worker-1.kubeconfig
worker-2.kubeconfig
```

### The kube-proxy Kubernetes Configuration File

Generate a kubeconfig file for the `kube-proxy` service:

```
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
  --kubeconfig=kube-proxy.kubeconfig
```

```
kubectl config set-credentials kube-proxy \
  --client-certificate=kube-proxy.pem \
  --client-key=kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig
```

```
kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig
```

```
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

## Distribute the Kubernetes Configuration Files

Copy the appropriate `kubelet` and `kube-proxy` kubeconfig files to each worker instance:

```
for instance in worker-1 ; do

  PRIVATE_HOSTNAME=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}"  \
    --query 'Reservations[].Instances[].[Tags[?Key==`PrivateHostName`].Value]' --output text)  

  PUBLIC_IP_ADDRESS=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" | \
    jq -r '.Reservations[].Instances[].PublicIpAddress')  
  scp -i ~/.ssh/kubernetes_the_hard_way -o "StrictHostKeyChecking no"  ${PRIVATE_HOSTNAME}.kubeconfig kube-proxy.kubeconfig \
    ubuntu@${PUBLIC_IP_ADDRESS}:~/
done
```

Next: [Generating the Data Encryption Config and Key](06-data-encryption-keys.md)
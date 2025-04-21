## 1. Helm Installation
Helm is used for managing Kubernetes applications. Install Helm using:
```sh
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```
**Resource**: [Helm Documentation](https://helm.sh/docs/intro/install/)

---

## 2. Install AWS EBS CSI Drivers
To manage persistent storage for Kubernetes:
1. Add the Helm repository for AWS EBS CSI Driver:
   ```sh
   helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
   helm repo update
   ```
2. Install the AWS EBS CSI Driver:
   ```sh
   helm upgrade --install aws-ebs-csi-driver \
       --namespace kube-system \
       aws-ebs-csi-driver/aws-ebs-csi-driver
   ```
**Resource**: [AWS EBS CSI Driver GitHub](https://github.com/kubernetes-sigs/aws-ebs-csi-driver/blob/master/docs/install.md)

---

## 3. Attach IAM Policies to Node Role
Ensure the following IAM policies are attached to the node role:
- `AmazonEBSCSIDriverPolicy`
- `AmazonEC2FullAccess`

---

## 4. Create a Storage Class
Apply the storage class configuration for AWS EBS:
```sh
kubectl create -f ebs-sc.yaml
```

---

## 5. Create Namespace
Create a dedicated namespace for the Robokart project:
```sh
kubectl apply -f namespace.yaml
```

---

## 6. Set the Default Namespace
Use `kubectx` and `kubens` for easier namespace management:
1. Install `kubectx` and `kubens`:
   ```sh
   sudo git clone https://github.com/ahmetb/kubectx /opt/kubectx
   sudo ln -s /opt/kubectx/kubectx /usr/local/bin/kubectx
   sudo ln -s /opt/kubectx/kubens /usr/local/bin/kubens
   ```
2. Switch to the `fusioniq` namespace:
   ```sh
   kubens fusioniq
   ```
**Resource**: [kubectx Repository](https://github.com/ahmetb/kubectx)

---

## 7.  install metrics server
```sh 
   kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
### To know resouces consume in pods
```
   kubectl top pods
```
```
   kubectl get pods -n robokart
```


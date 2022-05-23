# Prerequisite installations
- kubectl
- terraform
- aws
- git

# Initial cluster setup

### Provisioning initial kubernetes EKS cluster
```
cd terraform
terraform init
terraform apply -auto-approve
```

### Create kube config to be able to interact with cluster
```
aws eks --region $(terraform output -raw region) update-kubeconfig --name $(terraform output -raw cluster_name)
```

### Get cluster token
```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep service-controller-token | awk '{print $1}')
```
This token will be used for authenticating with the metrics dashboard if you wish to view it.
Output should look something like this:
```
Name:         service-controller-token-vtxc4
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: service-controller
              kubernetes.io/service-account.uid: 168a7594-b465-4f10-949e-91b93d60030e

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1066 bytes
namespace:  11 bytes
token:      <REDACTED>
```

### Applying the metrics-server
```
wget -O v0.3.6.tar.gz https://codeload.github.com/kubernetes-sigs/metrics-server/tar.gz/v0.3.6 && tar -xzf v0.3.6.tar.gz
kubectl apply -f metrics-server-0.3.6/deploy/1.8+/
rm -rf metrics-server-0.3.6
```

### Provision metrics dashboard UI
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta8/aio/deploy/recommended.yaml
```

### (Optional) To view dashboard, use kubectl to proxy bind to the internal http port
```
kubectl proxy
```
In browser visit: http://127.0.0.1:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
- select token radio button
- input above secret token that was generated
- submit

### Create kubernetes namespace for application deployments
```
kubectl create namespace staging
```

Everything up to this point only has to be run once. All steps after can be kicked off via Circle CI by pushing a commit to this repository.

### Apply application deployment
```
kubectl -n staging apply -f k8s/deployment.yaml
```

That is all that is needed to get the cluster, node, container, pods, loadbalancer, hpa, ingress, and everything else up and running. There is a delay in the metrics-server catching up to the provisioned application in regards to resource and loadbalancer resource. Both of these will be available within a few minutes of `apply` commands being run.

## Other notes
I've noticed that the replica set scaling through the deployment doesnt always take, I will need to look into the rollout properties as to what is going on here. If you run:
`kubectl -n staging scale deployment deployment/hello-world --replicas=8` it may not scale to 8, it can be done by editing the deployment, spec.replicas value to desired value in case that happens.

## In case of rollback
There isn't a proper versioning pattern, but if there were we could use this command to adjust the replica set images with one command `kubectl -n staging set image deployment/hello-world nginx=nginx:1.9.1`. You can modify the version to see the change occur for any other nginx version.


### For curling from outside the cluster
  - kubectl -n staging get services
  - grab "EXTERNAL-IP" for Type "LoadBalancer", it should be a long alphanumeric AWS url.
  - place the long name below in place of loadbalancer ingress, port is the defined port, 8765:
    - curl <LoadBalancer Ingress>:8765

### Other metrics
```
kubectl -n staging top pods
```

# CBSD + Minikube + Ansible AWX

This article walks through running AWX inside a Minikube virtual machine.
We’re assuming you’ve already set up Minikube using one of the two methods described here: [CBSD + Minikube](cbsd_minikube.md)

<center><img src="https://convectix.com/img/cbsd_minikube_aws2.png" width="1024" title="cbsd_minikube_aws2" alt="cbsd_minikube_aws2"/></center>

## AWS Operator

a) First, let's create a dedicated namespace for AWX and switch our context to it:

```
kubectl create namespace awx
```

There are several ways to deploy: using make deploy from the awx-operator repo, via Helm chart, or via kustomization.yaml.

<details>
  <summary>Options 1: kustomization.yaml</summary>

Here is the kustomization.yaml example:

:bangbang: | :information_source: Check for the latest tag (e.g., 2.19.1) here: https://github.com/ansible/awx-operator/releases
:---: | :---

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - github.com/ansible/awx-operator/config/default?ref=2.19.1

# Set the image tags to match the git version from above
images:
  - name: quay.io/ansible/awx-operator
    newTag: 2.19.1

# Specify a custom namespace in which to install AWX
namespace: awx
```

```
kubectl apply -k .
```

</details>


<details>
  <summary>Options 2: Helm chart</summary>

```
helm repo add awx-operator https://ansible-community.github.io/awx-operator-helm/
helm repo update
helm install awx-operator awx-operator/awx-operator -n awx --create-namespace
```

</details>

c) Verify that everything is in a Ready/Running state:

```
kubectl get pods -n awx
```

In my case with kustomization.yaml (because one does not simply deploy an app on Linux without a hiccup!), I hit an error where the pod couldn't pull the image from the `brancz` repo:

```
minikube ssh "docker pull brancz/kube-rbac-proxy:v0.15.0"
Error response from daemon: pull access denied for brancz/kube-rbac-proxy, repository does not exist or may require 'docker login': denied: requested access to the resource is denied
ssh: Process exited with status 1
```

To fix this, swap the repository:
```
kubectl edit deployment awx-operator-controller-manager -n awx
```

Replace gcr.io/kubebuilder/kube-rbac-proxy:v0.15.0 with quay.io/brancz/kube-rbac-proxy:v0.15.0. Save and exit (:wq), then check again after a few seconds:

```
kubectl get pods -n awx
```

You should see something like this:
```
NAME READY STATUS RESTARTS AGE
awx-operator-controller-manager-55c68ccbd-tld4f 2/2 Running 0 22s
```

The Operator is now ready.

## AWS Service

1) Create the configuration and launch the AWX instance using `awx-instance.yaml`:
```
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx-demo
  namespace: awx
spec:
  # Используем NodePort для простого доступа в Minikube
  service_type: nodeport
  # Quotas (optional)
#  postgres_resource_requirements:
#    requests:
#      cpu: 100m
#      memory: 256Mi
#    limits:
#      cpu: 500m
#      memory: 512Mi
```

```
kubectl apply -f awx-instance.yaml -n awx
```

Wait a bit (this usually takes 3–10 minutes):

```
kubectl get pods -n awx -w
```

Other helpful logs to monitor:
```
kubectl logs -f deployments/awx-operator-controller-manager -c awx-manager -n awx
```


2) Retrieve the admin password (the default login is admin):

```
kubectl get secret awx-demo-admin-password -o jsonpath="{.data.password}" -n awx | base64 --decode; echo
```

3) Get the login URL:
```
minikube service awx-demo-service --url -n awx
```

4) (Optional) If accessing from an external network, proxy port 8080 to the service:
```
kubectl port-forward --address 0.0.0.0 svc/awx-demo-service -n awx 8080:80
```

Now you can open the UI and log in using admin/TOKEN. Happy automating!

<center><img src="https://convectix.com/img/cbsd_minikube_aws1.png" width="1024" title="cbsd_minikube_aws1" alt="cbsd_minikube_aws1"/></center>


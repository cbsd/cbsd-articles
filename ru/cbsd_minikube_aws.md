# CBSD + Minikube + Ansible AWX

В этой статье описывается вариант запуска AWK с использования minikube виртуальной машины.

Предполагаем, что вы развернули minikube одним из двух предлагаемых способов из этой статьи: [CBSD + Minikube](cbsd_minikube.md)

<center><img src="https://convectix.com/img/cbsd_minikube_aws2.png" width="1024" title="cbsd_minikube_aws2" alt="cbsd_minikube_aws2"/></center>

## AWS Operator

a) Создадим для AWX отдельный namespace и будем работать исключительно в нем:

```
kubectl create namespace awx
```

b) Есть несколько способов деплоя, `make deploy` в https://github.com/ansible/awx-operator.git, Helm chart или через kustomization.yaml, выберем его. Написал следующий kustomization.yaml:

:bangbang: | :information_source: Актуальный тег (2.19.1 в данном примере) смотрите тут: https://github.com/ansible/awx-operator/releases
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

Или helm chart:

```
helm repo add awx-operator https://ansible-community.github.io/awx-operator-helm/
helm repo update
helm install awx-operator awx-operator/awx-operator -n awx --create-namespace
```

c) Проверка что все в статусе Ready/running:

```
kubectl get pods -n awx
```

В моем случае (нельзя на Linux просто так взять и просто продеплоить приложение!) возникла ошибка с тем, что под не скачивается с `brancz`:
```
minikube ssh "docker pull brancz/kube-rbac-proxy:v0.15.0"
Error response from daemon: pull access denied for brancz/kube-rbac-proxy, repository does not exist or may require 'docker login': denied: requested access to the resource is denied
ssh: Process exited with status 1
```

Меняем репу на другую:
```
kubectl edit deployment awx-operator-controller-manager -n awx
```

Заменяя `gcr.io/kubebuilder/kube-rbac-proxy:v0.15.0` на `quay.io/brancz/kube-rbac-proxy:v0.15.0`, после `:wq` через пару секунд проверить опять:
```
kubectl get pods -n awx
```
Должно быть что-то вроде:
```
NAME READY STATUS RESTARTS AGE
awx-operator-controller-manager-55c68ccbd-tld4f 2/2 Running 0 22s
```

Оператор готов.

## AWS Service

1) Создаем конфигурацию и запускаем экземпляр AWS, awx-instance.yaml:
```
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx-demo
  namespace: awx
spec:
  # Используем NodePort для простого доступа в Minikube
  service_type: nodeport
  # Настройки ресурсов (опционально, но полезно для Minikube)
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

Ждем (может занять 3-10 минут):

```
kubectl get pods -n awx -w
```

другие полезные логи:
```
kubectl logs -f deployments/awx-operator-controller-manager -c awx-manager -n awx
```


2) Узнать пароль администратора (логин по умолчанию admin):

```
kubectl get secret awx-demo-admin-password -o jsonpath="{.data.password}" -n awx | base64 --decode; echo
```

3) Узнать URL для входа:
```
minikube service awx-demo-service --url -n awx
```

4) (опционально, если заходить извне) - спроксировать `:8080` порт на сервис:
```
kubectl port-forward --address 0.0.0.0 svc/awx-demo-service -n awx 8080:80
```

После чего можно обратиться на UI и воспользоваться `admin/TOKEN` для входа. Приятной работы!

<center><img src="https://convectix.com/img/cbsd_minikube_aws1.png" width="1024" title="cbsd_minikube_aws1" alt="cbsd_minikube_aws1"/></center>


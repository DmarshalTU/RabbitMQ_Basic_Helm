# RabbitMQ_Basic_Helm and Installing RabbitMQ Cluster Operator

## Installation using Helm chart:

* add bitnami repo to helm repo list

    ```bash
    ~ helm repo add bitnami https://charts.bitnami.com/bitnami

    "bitnami" already exists with the same configuration, skipping
    ```

* install rabbitmq-operator

    ```bash
    helm install rabbitmq-uat bitnami/rabbitmq-cluster-operator --namespace rabbitmq
    ```

    output:
    ```bash
    NAME: rabbitmq
    LAST DEPLOYED: Wed Mar 30 11:18:52 2022
    NAMESPACE: rabbitmq
    STATUS: deployed
    REVISION: 1
    TEST SUITE: None
    NOTES:
    CHART NAME: rabbitmq-cluster-operator
    CHART VERSION: 2.5.0
    APP VERSION: 1.12.1

    ** Please be patient while the chart is being deployed **

    Watch the RabbitMQ Cluster Operator and RabbitMQ Messaging Topology Operator Deployment status using the command:

    kubectl get deploy -w --namespace rabbitmq -l app.kubernetes.io/name=rabbitmq-cluster-operator,app.kubernetes.io/instance=rabbitmq
    ```

* login to web dashboard

    ```bash
    kubectl port-forward "service/definition" 15672 -n rabbitmq
    ```

    and open http://localhost:15672 on your browser

## Create a RabbitMQ Instance
To create a RabbitMQ instance, a **RabbitmqCluster** resource definition must be created and applied. *RabbitMQ Cluster Kubernetes Operator creates the necessary resources*, such as Services and StatefulSet, in the same namespace in which the **RabbitmqCluster** was defined.

First, create a YAML file to define a RabbitmqCluster resource named definition.yaml.

```yaml
---
apiVersion: rabbitmq.com/v1beta1
kind: RabbitmqCluster
metadata:
  name: rabbitmq-dev
spec:
  replicas: 1
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 800m
      memory: 1Gi
  service:
    type: LoadBalancer
---
apiVersion: rabbitmq.com/v1beta1
kind: Vhost
metadata:
  name: test # name of this custom resource
  namespace: rabbitmq
spec:
  name: TEST # name of the vhost
  rabbitmqClusterReference:
    name: rabbitmq
---
apiVersion: v1
kind: Secret
metadata:
  name: rabbitmq-user-cred
type: Opaque
stringData:
  username: denis
  password: denis1
---
apiVersion: rabbitmq.com/v1beta1
kind: User
metadata:
  name: rabbitmq-user
spec:
  tags: # available tags are 'management', 'policymaker', 'monitoring' and 'administrator'
  - administrator
  - management
  - policymaker
  - monitoring

  rabbitmqClusterReference:
    name: rabbitmq-dev
  importCredentialsSecret:
    name: rabbitmq-user-cred
---
apiVersion: rabbitmq.com/v1beta1
kind: Permission
metadata:
  name: rabbitmq-user
spec:
  vhost: "TEST"
  user: "rabbitmq-user" # name corresponds to the username we provided in "rabbitmq-dev-user-cred" secret
  permissions:
    write: ".*"
    configure: ""
    read: ".*"
  rabbitmqClusterReference:
    name: rabbitmq
```

```bash
kubectl run perf-test --image=pivotalrabbitmq/perf-test -n rabbitmq -- --uri "amqp://default_user:password@definition"

kubectl logs -f perf-test -n rabbitmq

kubectl port-forward "service/definition" 15672 -n rabbitmq

kubectl expose deployment deployment-rabbitmq --type=LoadBalancer --name=rabbitmq-proxy --namespace rabbitmq --port 15672
```

``` yaml
  rabbitmq:
    additionalConfig: |
      log.console.level = info
      channel_max = 1700
      default_user= devopsn 
      default_pass = Lastpass1
      default_user_tags.administrator = true


  override:
    service:
      spec:
          ports:
            - name: web-ui
              protocol: TCP
              port: 80
              targetPort: 15672
```

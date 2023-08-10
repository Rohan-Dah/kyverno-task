
# Kyverno: Policy Exceptions 2.0 Tasks

### 1) Write a Kyverno policy that limits the amount of containers that can be in a single Pod. The policy should check all pods to ensure that they have no more than two containers. You should show a case where a resource is blocked due to policy violation and another case where a resource is successfully created.

### Step 1: Setting up minikube for kubernetes cluster
Before deploying kyverno, we need to create a local kubernetes cluster using minikube. I followed the below documentation for the same:

https://minikube.sigs.k8s.io/docs/start/ 

To make sure PolicyExceptions are enabled copy paste the content from the below file in a file called values.yaml
https://github.com/kyverno/kyverno/blob/v1.10.2/charts/kyverno/values.yaml#L390

Make sure that field PolicyExceptions Enabled is set to true

### Step 2: Creating the cluster:
For creating a local cluster:
```bash
  minikube start
```

Verifying the cluster:
```bash
  kubectl cluster-info
```
The output I got:
```bash
Kubernetes control plane is running at https://192.168.49.2:8443
CoreDNS is running at https://192.168.49.2:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

Hence the local cluster is up and running.

### Step 3: Deploying kyverno in the kubernetes cluster
We can deploy kyverno in the local cluster by following the below steps:

```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
```
Install using the below command after setting up values.yaml

```bash
helm install kyverno kyverno/kyverno -n kyverno --create-namespace -f values.yaml
```

Verifying kyverno:

```bash
  kubectl get ns
```
Output:
```bash
  NAME              STATUS   AGE
default           Active   131m
delta             Active   127m
kube-node-lease   Active   131m
kube-public       Active   131m
kube-system       Active   131m
kyverno           Active   94m
```

Hence kyverno is set up

### Step 4: Policy for limiting containers in a pod
 
```bash
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: limit-containers-per-pod
  annotations:
    policies.kyverno.io/title: Limit Containers per Pod
    policies.kyverno.io/category: Sample
    policies.kyverno.io/minversion: 1.3.6
    policies.kyverno.io/subject: Pod
    policies.kyverno.io/description: >-
      Pods can have many different containers which
      are tightly coupled. It may be desirable to limit the amount of containers that
      can be in a single Pod to control best practice application or so policy can
      be applied consistently. This policy checks all Pods to ensure they have
      no more than two containers.
spec:
  validationFailureAction: Enforce
  background: false
  rules:
  - name: limit-containers-per-pod-controllers
    match:
      resources:
        kinds:
        - Deployment
        - DaemonSet
        - Job
        - StatefulSet
    preconditions:
      all:
      - key: "{{request.operation}}"
        operator: Equal
        value: CREATE
    validate:
      message: "Pods can only have a maximum of 2 containers."
      deny:
        conditions:
          any:
          - key: "{{request.object.spec.template.spec.containers[] | length(@)}}"
            operator: GreaterThan
            value: "2"
  - name: limit-containers-per-pod-bare
    match:
      resources:
        kinds:
        - Pod
    preconditions:
      all:
      - key: "{{request.operation}}"
        operator: Equal
        value: CREATE
    validate:
      message: "Pods can only have a maximum of 2 containers."
      deny:
        conditions:
          any:
          - key: "{{request.object.spec.containers[] | length(@)}}"
            operator: GreaterThan
            value: "2"
  - name: limit-containers-per-pod-cronjob
    match:
      resources:
        kinds:
        - CronJob
    preconditions:
      all:
      - key: "{{request.operation}}"
        operator: Equal
        value: CREATE
    validate:
      message: "Pods can only have a maximum of 2 containers."
      deny:
        conditions:
          any:
          - key: "{{request.object.spec.jobTemplate.spec.template.spec.containers[] | length(@)}}"
            operator: GreaterThan
            value: "2"

```
Store the above policy in:

```bash
  limit-containers-policy.yaml
```

The Enforce option is responsible to block any resource that violates the policy.

Apply the above policy:
```bash
  kubectl apply -f limit-containers-policy.yaml
```



### Step 5: Verifying the policy

Firstly we need to create a pod that violates the above policy:

```bash
apiVersion: v1
kind: Pod
metadata:
  name: violating-pod
spec:
  containers:
  - name: nginx
    image: nginx:latest
  - name: redis
    image: redis:latest
  - name: postgres #Violating pod
    image: postgres:latest
```

Here we created have created a pod with three container that violates the policy. 

```bash
  kubectl apply -f violating-pod.yaml
```

Output:
```bash
  Error from server: error when creating "violating-pod.yaml": admission webhook   "validate.kyverno.svc-fail" denied the request: 

  resource Pod/default/violating-pod was blocked due to the following policies 

  limit-containers-per-pod:
  limit-containers-per-pod-bare: Pods can only have a maximum of 2 containers.
```

The output states, the pod is being blocked because it violates the policy.

Now, lets try a pod that complies with the policy to check whether it is being created or not.

```bash
apiVersion: v1
kind: Pod
metadata:
  name: compliant-pod
spec:
  containers:
    - name: container1
      image: nginx:latest
    - name: container2
      image: busybox:latest 
```

The above pod has two containers and it should not violate the policy and hence pod should be created.

```bash
kubectl apply -f compliant-pod.yaml
```

Output:
```bash
pod/compliant-pod created
```

Hence the policy is working..





### 2) Use the policy you created in the 1st step to write a Kyverno policy exception that grants an exception to a Pod or Deployment named important-app which will be created in the delta namespace. You should show a case where a resource is successfully created although it violates the policy.

#### Step 1: Create a policy exception


```bash
apiVersion: kyverno.io/v2alpha1
kind: PolicyException
metadata:
  name: delta-exception
  namespace: delta
spec:
  exceptions:
  - policyName: limit-containers-per-pod
    ruleNames:
    - limit-containers-per-pod-bare
  match:
    any:
    - resources:
        kinds:
        - Pod
        - Deployment
        namespaces:
        - delta
        names:
        - important-app*
```

Apply the above above policy exception:
```bash
kubectl apply -f delta-exception.yaml
```
Output:

```bash
policyexception.kyverno.io/delta-exception created
```

The above policy exception states that, if we have a pod or deployment named important-app, it will be created if it is a delta namespace resource, which in the above case is true.

#### Step 2: Create a pod named important-app.yaml that is set to delta namespace

```bash 
apiVersion: v1
kind: Pod
metadata:
  name: important-app
  namespace: delta
spec:
  containers:
  - name: nginx
    image: nginx
  - name: redis
    image: redis:latest
  - name: postgres
    image: postgres:latest
```

```bash
kubectl apply -f important-app.yaml
```
Output
```bash 
pod/important-app created
```

As we can see, the above pod named important-app has three containers that must violates the policy, but due to policyexception it is not.

Thank you









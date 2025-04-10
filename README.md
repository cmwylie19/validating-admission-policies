# Validating Admission Policies

Validating admission policies offer a declaritive, in-process alternative to validating admission webhooks using Common Expression Language (CEL).

**TOC**
- [Validating Admission Policies](#validating-admission-policies)
  - [How to use them](#how-to-use-them)
  - [Example](#example)
  - [Usage](#usage)
  - [Using variables](#using-variables)
  - [Closing](#closing)

## How to use them

Policies are made up of three resources:
- `ValidatingAdmissionPolicy` - describes the abstract logic of a policy.
- `ValidatingAdmissionPolicyBinding` - Links the policy to a set of resources for scoping.

## Example

This example shows how to create a policy that only allows the `kubernetes` Service, Endpoint, or EndpointSlice to be created/modified in the `default` namespace. **A validation passes (request is allowed) when the expression returns true.**

```yaml
kubectl apply -f -<<EOF
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: "default-ns-policy.defenseunicorns.com"
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups:   ["*"]
      apiVersions: ["*"]
      operations:  ["*"]
      resources:   ["*"]
  validations:
    - expression: "object.kind in ['Service', 'Endpoint', 'EndpointSlice'] && object.metadata.name == 'kubernetes'"
      message: "Only the 'kubernetes' Service, Endpoint, or EndpointSlice is allowed. All other changes are denied."
EOF
```

```yaml
kubectl apply -f -<<EOF
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: "default-ns-policy-binding.defenseunicorns.com"
spec:
  policyName: "default-ns-policy.defenseunicorns.com"
  validationActions: [Deny]
  matchResources:
    namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: default
EOF
```

### Usage

Create a pod

```yaml
> k run hi --image=nginx
The pods "hi" is invalid: : ValidatingAdmissionPolicy 'default-ns-policy.defenseunicorns.com' with binding 'default-ns-policy-binding.defenseunicorns.com' denied request: Only the 'kubernetes' Service, Endpoint, or EndpointSlice is allowed. All other changes are denied.
```

Create a role

```yaml
> k create role pod-reader --verb=get --verb=list --verb=watch --resource=pods 
The roles "pod-reader" is invalid: : ValidatingAdmissionPolicy 'default-ns-policy.defenseunicorns.com' with binding 'default-ns-policy-binding.defenseunicorns.com' denied request: Only the 'kubernetes' Service, Endpoint, or EndpointSlice is allowed. All other changes are denied.
```

Create a service account

```yaml
> k create sa test
error: failed to create serviceaccount: serviceaccounts "test" is forbidden: ValidatingAdmissionPolicy 'default-ns-policy.defenseunicorns.com' with binding 'default-ns-policy-binding.defenseunicorns.com' denied request: Only the 'kubernetes' Service, Endpoint, or EndpointSlice is allowed. All other changes are denied.
```

Update the Kubernetes Service

```yaml
> k label svc/kubernetes color=blue
service/kubernetes labeled
```

## Using variables

We can define variables and use them in validations. For instance, if we do not want to allow pods or containers running below UID 1000, we can use the following policy and define variables for pod security context and containers.

```yaml
kubectl apply -f -<<EOF
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: "security-contexts.defenseunicorns.com"
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups:   [""]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["pods"]
  variables: # define the variable
    - name: "podSecurityContext"
      expression: "object.spec.securityContext"
  validations:
    - expression: "has(variables.podSecurityContext.runAsUser) ? variables.podSecurityContext.runAsUser >= 1000 : true"
      message: "Pod security context must run as user 1000 or higher."
    - expression: "object.spec.containers.all(c, has(c.securityContext) && has(c.securityContext.runAsUser) ? c.securityContext.runAsUser >= 1000 : true)"
      message: "All containers must run as user 1000 or higher if set."
    - expression: "has(object.spec.initContainers) ? object.spec.initContainers.all(c, has(c.securityContext) && has(c.securityContext.runAsUser) ? c.securityContext.runAsUser >= 1000 : true) : true"
      message: "All initContainers must run as user 1000 or higher if set."
    - expression: "has(object.spec.ephemeralContainers) ? object.spec.ephemeralContainers.all(c, has(c.securityContext) && has(c.securityContext.runAsUser) ? c.securityContext.runAsUser >= 1000 : true) : true"
      message: "All ephemeralContainers must run as user 1000 or higher if set."
EOF
```

```yaml
kubectl apply -f -<<EOF
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: "security-contexts-binding.defenseunicorns.com"
spec:
  policyName: "security-contexts.defenseunicorns.com"
  validationActions: [Deny]
  matchResources:
    namespaceSelector:
      matchExpressions:
        - key: kubernetes.io/metadata.name
          operator: NotIn
          values: ["kube-system"]
EOF
```

Create a pod that violates the policy due to  securityContext.


```yaml
kubectl apply -f -<<EOF
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: violation
  name: violation
  namespace: kube-public
spec:
  securityContext:
    runAsUser: 999
  containers:
  - image: nginx
    name: test
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
EOF

The pods "violation" is invalid: : ValidatingAdmissionPolicy 'security-contexts.defenseunicorns.com' with binding 'security-contexts-binding.defenseunicorns.com' denied request: Pod security context must run as user 1000 or higher.
```

This will fail pod security context check as the user is set to 999


```yaml
kubectl apply -f -<<EOF
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: violation
  name: violation
  namespace: kube-public
spec:
  containers:
  - image: nginx
    name: test
    resources: {}
    securityContext:
      runAsUser: 100
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
EOF

The pods "violation" is invalid: : ValidatingAdmissionPolicy 'security-contexts.defenseunicorns.com' with binding 'security-contexts-binding.defenseunicorns.com' denied request: All containers must run as user 1000 or higher if set.
```

Now lets use an init container with a violation

```yaml
kubectl apply -f -<<EOF
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: violation
  name: violation
  namespace: kube-public
spec:
  initContainers:
  - image: busybox
    name: init
    command: ['sh', '-c', 'sleep 10']
    resources: {}
    securityContext:
      runAsUser: 999
  containers:
  - image: nginx
    name: test
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
EOF

The pods "violation" is invalid: : ValidatingAdmissionPolicy 'security-contexts.defenseunicorns.com' with binding 'security-contexts-binding.defenseunicorns.com' denied request: All initContainers must run as user 1000 or higher if set.
```

Now create a pod that passes the expressions

```yaml
kubectl apply -f -<<EOF
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: violation
  name: violation
  namespace: kube-public
spec:
  initContainers:
  - image: busybox
    name: init
    command: ['sh', '-c', 'sleep 10']
    resources: {}
    securityContext:
      runAsUser: 1000
  containers:
  - image: nginx
    name: test
    resources: {}
    securityContext:
      runAsUser: 1000
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
EOF
```

## Closing

It is a powerful tool that can be used to enforce policies and ensure compliance in your Kubernetes cluster. However, it is important to understand the implications of using it and to test your policies thoroughly before deploying them in a production environment. This is a good solution for single policies but for complex logic around Admission Events you still are going to need an operator.

- [Docs](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/)
- [CEL Playground](https://playcel.undistro.io/)

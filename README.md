# Validating Admission Policies

Validating admission policies offer a declaritive, in-process alternative to validating admission webhooks using Common Expression Language (CEL).

**TOC**
- [Validating Admission Policies](#validating-admission-policies)
  - [How to use them](#how-to-use-them)
  - [Example](#example)
  - [Installation](#installation)
  - [Usage](#usage)
  - [License](#license)

## How to use them

Policies are made up of three resources:
- `ValidatingAdmissionPolicy` - describes the abstract logic of a policy.
- `ValidatingAdmissionPolicyBinding` - Links the policy to a set of resources for scoping.

## Example

This example shows how to create a policy that only allows the `kubernetes` Service, Endpoint, or EndpointSlice to be created/modified in the `default` namespace.

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

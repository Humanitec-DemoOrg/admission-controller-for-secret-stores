# admission-controller-for-secret-stores

TOC:
- [Create the policy](#create-the-policy)
- [Test without Humanitec Deployment](#test-without-humanitec-deployment)
- [Test with Humanitec Deployment](#test-with-humanitec-deployment)
- [Thoughts and next steps](#thoughts-and-next-steps)

## Create the policy

We are using [`ValidatingAdmissionPolicy`](https://medium.com/p/ed1321bcf739), built-in in Kubernetes, even if this could be achieved with other admission controllers like Kyverno or OPA Gatekeeper.

Deploy the policy in your cluster:
```bash
kubectl apply -f secretmappings-policy.yaml
```

```bash
kubectl get ValidatingAdmissionPolicy check-secret-store-access-for-secretmappings
kubectl get ValidatingAdmissionPolicyBinding check-secret-store-access-for-secretmappings
```

## Test without Humanitec Deployment

```bash
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: check-secret-store-access-for-secretmappings
  namespace: default
data:
  allowedSecretStoreNames: "default,valid"
EOF
```

Test with invalid `SecretMapping`:
```bash
cat << EOF | kubectl apply -f -
apiVersion: humanitec.io/v1alpha1
kind: SecretMapping
metadata:
  name: invalid-secretstore
  namespace: default
spec:
  secretData:
  - remoteRef:
      key: my-secret-shared-value
      version: ""
    secretKey: my-secret-shared-value
    secretStoreName: invalid
  secretName: shared-secrets
EOF
```

You'll get this error message:
```bash
The secretmappings "invalid-secretstore" is invalid: : ValidatingAdmissionPolicy 'check-secret-store-access' with binding 'check-secret-store-access' denied request: secretStoreName is not allowed in this list:default,valid
```

Test with valid `SecretMapping`:
```bash
cat << EOF | kubectl apply -f -
apiVersion: humanitec.io/v1alpha1
kind: SecretMapping
metadata:
  name: valid-secretstore
  namespace: default
spec:
  secretData:
  - remoteRef:
      key: my-secret-shared-value
      version: ""
    secretKey: my-secret-shared-value
    secretStoreName: default
  secretName: shared-secrets
EOF
```

## Test with Humanitec Deployment

Configure a `base-env` configuring the associated `ConfigMap` as parameter of the policy:
```bash
humctl apply -f base-env.yaml

humctl apply -f default-secretstore-config.yaml
```
_Note: at this stage, we just create one global/default `config` with the `SecretStore` names allowed. You can create as many and granular `config` as you want._

Test with Humanitec invalid Shared Values&Secrets:
```bash
APP_ID=FIXME

humctl create value invalid invalid \
  --app ${APP_ID} \
  --is-secret-ref \
  --secret-store invalid
```

Deploy a Workload.

You'll see that your Deployment is failing with this error message:
```none
Error applying manifests to cluster
Applying manifest to cluster: The secretmappings "invalid-secretstore" is invalid: : ValidatingAdmissionPolicy 'check-secret-store-access' with binding 'check-secret-store-access' denied request: secretStoreName is not allowed in this list:default,valid
```

```bash
APP_ID=FIXME

humctl create value valid valid \
  --app ${APP_ID} \
  --is-secret-ref \
  --secret-store default
```

Deploy a Workload.

You'll see that your Deployment is successful.

## Thoughts and next steps


- This doesn't (yet) support `resources`, it's only focusing on `secretmappings` that can be directly used by Devs via the Shared Values&Secrets, as opposed to the `resources` only configurable by PEs.
- Instead of having a `ConfigMap` as `param` for the policy, depending on use cases, an idea would be to use the `SecretStore` itself in the `humanitec-operator[-system]` namespace, if it's only one `SecretStore` per namespace.
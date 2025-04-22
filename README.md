# admission-controller-for-secret-stores

```bash
kubectl apply -f policy.yaml
```

```bash
kubectl get ValidatingAdmissionPolicy check-secret-store-access-for-secretmappings
kubectl get ValidatingAdmissionPolicyBinding check-secret-store-access-for-secretmappings
```

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
# Create a PKI using Smallstep and Cert-Manager

## Install PKI

```bash
helm repo add smallstep https://smallstep.github.io/helm-charts/
helm repo update
helm install -f values.yaml -n step-ca --create-namespace \
     --set inject.secrets.ca_password=$(cat password.txt) \
     --set inject.secrets.provisioner_password=$(cat password.txt) \
     --set service.targetPort=9000 \
     step-certificates smallstep/step-certificates
```

```bash
helm install step-issuer smallstep/step-issuer -n step-issuer
```

## Install cert-manager

```bash
helm install cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --version v1.19.2 \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```

## Create Step Issuer

```yaml
apiVersion: certmanager.step.sm/v1beta1
kind: StepIssuer
metadata:
  name: step-issuer
  namespace: step-issuer
spec:
  # The CA URL.
  url: https://step-certificates.step-ca.svc.cluster.local
  # The base64 encoded version of the CA root certificate in PEM format.
  caBundle: |
    LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUJvekNDQVVtZ0F3SUJBZ0lRVGVoejUxandX
    Y0FzdmpVSXNORWtBakFLQmdncWhrak9QUVFEQWpBd01SSXcKRUFZRFZRUUtFd2xUYldGc2JITjBa
    WEF4R2pBWUJnTlZCQU1URVZOdFlXeHNjM1JsY0NCU2IyOTBJRU5CTUI0WApEVEkyTURJd05URXpO
    VFV4TUZvWERUTTJNREl3TXpFek5UVXhNRm93TURFU01CQUdBMVVFQ2hNSlUyMWhiR3h6CmRHVndN
    Um93R0FZRFZRUURFeEZUYldGc2JITjBaWEFnVW05dmRDQkRRVEJaTUJNR0J5cUdTTTQ5QWdFR0ND
    cUcKU000OUF3RUhBMElBQkt0YjRtYVJkZW9mZEZscmRzQzEwT2dpaUhNSmhJRFg5YVZMODZ4Y1FL
    WVV5djBMdkxCVwpEcEtJQmZXZUZ2Rzk4L2FOVEVBaTlZOTFPR2cxTkhIS3k3NmpSVEJETUE0R0Ex
    VWREd0VCL3dRRUF3SUJCakFTCkJnTlZIUk1CQWY4RUNEQUdBUUgvQWdFQk1CMEdBMVVkRGdRV0JC
    UzFiYVdJN1psbUsrUU44UEhSaHk1R3dOd0IKTGpBS0JnZ3Foa2pPUFFRREFnTklBREJGQWlBWEZB
    VWFCZS9iWVFPUTdQcW14c3oyZWVDTEdGd3UwcEJ6eFZZOQpHZkYwM3dJaEFJdUxKOHRBaC83Q3V0
    K2JMUTN1VjFTcWlCVVAwNGMweDFlcU5lRjFtQ3l4Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K

  # The provisioner name, kid, and a reference to the provisioner password secret.
  provisioner:
    name: step-issuer
    kid:
    passwordRef:
      name: step-issuer-provisioner-password
      key: password
```

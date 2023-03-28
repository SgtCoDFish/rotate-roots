# In-cluster PKI

This example requires both cert-manager and approver-policy to be installed.

⚠️ An existing cert-manager installation might not work for approver-policy; see [the docs](https://cert-manager.io/docs/projects/approver-policy/)
for details on how to install properly.

## Requirements

Kubernetes cluster running up-to-date versions of the following:

- cert-manager (with default approver disabled)
- approver-policy
- trust-manager

Note that trust-manager's Helm chart includes a Certificate which you'll need to approve manually.

Installation instructions are given below as a guide:

```console
# if you already have a cluster, skip this step
kind create cluster --name ap

# install cert-manager with approver disabled
helm upgrade -i -n cert-manager cert-manager jetstack/cert-manager --set extraArgs={--controllers='*\,-certificaterequests-approver'} --set installCRDs=true --create-namespace

# install approver-policy
helm upgrade -i -n cert-manager cert-manager-approver-policy jetstack/cert-manager-approver-policy --wait

# install trust-manager
helm upgrade -i -n cert-manager trust-manager jetstack/trust-manager --wait

# approve trust-manager CertificateRequest (this actually approves all certs in the cert-manager namespace; you might want to be more careful in prod)
kubectl get -n cert-manager cr -o go-template="{{range .items}}{{printf \"%s\n\" .metadata.name}}{{end}}" | xargs -I% cmctl approve -n cert-manager %
```

## Using the Example: Issuance

The expected flow is as follows to demonstrate safe issuance using private PKI:

```console
# create our in-cluster root certificate
kubectl apply -f incluster.yaml

# approve the root certificate CertificateRequest (this actually approves all certs in the cert-manager namespace; you might want to be more careful in prod)
kubectl get -n cert-manager cr -o go-template="{{range .items}}{{printf \"%s\n\" .metadata.name}}{{end}}" | xargs -I% cmctl approve -n cert-manager %

# apply policy to prevent misuse of our issuer
kubectl apply -f policy.yaml

# create an example cert which should be approved and issued automatically
kubectl apply -f cert-example-allowed.yaml

# create an example cert which should be denied
kubectl apply -f cert-example-denied.yaml

# delete the denied cert after looking at it since it's just an example
kubectl delete -f cert-example-denied.yaml
```

## Using the Example: Trust

Our certs won't be able to be trusted until they're in a trust store, which we can use trust-manager for:

```console
# get the root certificate we created above and save it to a file
kubectl get -n cert-manager secrets root-secret -o go-template='{{index .data "tls.crt"}}' | base64 -d > myroot.pem

# create a new ConfigMap containing just that root certificate under the key "root.pem"
kubectl create configmap -n cert-manager cluster-root --from-file=root.pem=myroot.pem

# create a Bundle using that configmap along with the built-in trust bundle
kubectl apply -f bundle.yaml
```

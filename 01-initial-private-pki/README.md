# Initial Private PKI

In this section, we'll set up a full private PKI in our cluster which is ready to be used in production.

Importantly, though, you _can't_ stop here! This is a great start but you need to be able to do rotation to be able to run safely in the long term!

The next section after this one will help with rotation.

## Requirements

You should have installed all the prerequisites from the [repo README](../README.md).

## Creating Issuers and Policy

The expected flow is as follows to demonstrate safe issuance using private PKI:

```bash
# create our in-cluster root certificate
kubectl apply -f incluster.yaml

# approve the root certificate CertificateRequest (this actually approves all certs in the cert-manager namespace; you might want to be more careful in prod)
# you might see an error about a CertificateRequest already being approved - that's probably the one above, for trust-manager!
# as long as you see a message like "Approved CertificateRequest 'cert-manager/root-certificate-xxxxx'" you should be good to go
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

## Handling Trust

Our certs won't be able to be trusted until they're in a trust store, which is where trust-manager comes in!

Note that we don't directly use the Secret which holds the root certificate, instead creating a ConfigMap. This is important!

See the talk for more details on why.

```bash
# get the root certificate we created above and save it to a file
kubectl get -n cert-manager secrets root-secret -o go-template='{{index .data "tls.crt"}}' | base64 -d > myroot.pem

# create a new ConfigMap containing just that root certificate under the key "root.pem"
kubectl create configmap -n cert-manager cluster-root --from-file=root.pem=myroot.pem

# create a Bundle using that configmap along with the built-in trust bundle
kubectl apply -f bundle.yaml
```

## Success!

Now move onto [section 2](../02-safe-root-rotation/README.md)!

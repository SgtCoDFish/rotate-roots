# Safe Root Rotation

As mentioned in the talk, we can't avoid having a plan for rotation and in fact to do anything securely we need to be able to do it in an emergency!

In this section we'll carry out a rotation of our root certificate, safely and without downtime.

## Requirements

You should have installed all the prerequisites from the [repo README](../README.md).

[Section 1](../01-initial-private-pki/README.md) should be completed. This example builds upon that section.

## Rotation Example

### Step 1: Create a New Root Certificate

We can't rotate our old cert without having something new to rotate in!

We can use the same `selfsigned-issuer` ClusterIssuer from step one to create our new root certificate, in a new Secret:

```console
# create the root, which will need to be approved
kubectl apply -f newroot.yaml

# approve the new CertificateRequest (this actually approves all certs in the cert-manager namespace; you might want to be more careful in prod)
kubectl get -n cert-manager cr -o go-template="{{range .items}}{{printf \"%s\n\" .metadata.name}}{{end}}" | xargs -I% cmctl approve -n cert-manager %
```

### Step 2: Trust the New Root

Without trusting the root it's useless, so we need to add it to our `Bundle`!

Note that - as in step 1 - we don't directly use the Secret which holds the root certificate, instead creating a ConfigMap. This is important! See the talk for more details on why.

```console
kubectl get -n cert-manager secrets root-secret-2 -o go-template='{{index .data "tls.crt"}}' | base64 -d > myroot2.pem

kubectl create configmap -n cert-manager cluster-root-2 --from-file=root.pem=myroot2.pem

kubectl apply -f bundle-update.yaml
```

### Step 3: Ensure Everything Uses the New Trust Bundle

In general, this step cannot be automated. Anywhere that your old CA is trusted, you'll need to update the bundle to include your new CA!

In this example, we're not directly using the trust store anyway - we're just updating the `Bundle` without consuming the resulting `ConfigMap`.

So this step is easy here, but don't expect it to be so easy in production!

### Step 4: Shift Issuance to the New Root

We only have one Certificate which we issued off the old root so we only need to update that one certificate, but again this could
be more tricky in a production environment.

```console
kubectl apply -f cert-example-updated.yaml
```

### Step 5: Remove the Old Root

This is simple enough - we simply update the `Bundle` one last time:

```console
kubectl apply -f bundle-final.yaml

# delete old root ConfigMap, just for cleanup
kubectl delete -n cert-manager configmap cluster-root
```

## Success!

We've just carried out a safe rotation of our root certificate!

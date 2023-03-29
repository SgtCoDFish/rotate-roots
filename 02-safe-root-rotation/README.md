# Safe Root Rotation

As mentioned in the talk, we can't avoid having a plan for rotation and in fact to do anything securely we need to be able to do it in an emergency!

In this section we'll carry out a rotation of our root certificate, safely and without downtime.

## Requirements

Section 1 - "Initial Private PKI" - should be completed. This example builds upon that example, so if you customised the first steps you'll
possibly need to customise this section as well.

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

### Step 4: Shift Issuance to the New Root

### Step 5: Remove the Old Root

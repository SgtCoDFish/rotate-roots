# Rotate Roots Right Round

This repo is a companion to my KubeCon EU 2023 talk, [Rotate Roots Right Round: Using cert-manager for Safer Private PKI](https://kccnceu2023.sched.com/event/1HyZ6).

It provides example configuration and steps for setting up private PKI in Kubernetes using:

1. cert-manager (of course!)
2. [approver-policy](https://cert-manager.io/docs/projects/approver-policy/)
3. and [trust-manager](https://cert-manager.io/docs/projects/trust-manager/)

Subfolders are numbered in order; follow them in numerical order.

Using this repo, along with the talk, you should be able to safely set up a private PKI that's free of cost, free of rate limits and which gives you complete control!

## Prerequisites

On your development machine, you'll need to have installed [cmctl](https://cert-manager.io/docs/reference/cmctl/).

You'll also need a Kubernetes cluster running up-to-date versions of the following:

- cert-manager (with default approver disabled)
- approver-policy
- trust-manager

⚠️ An existing cert-manager installation might not work for approver-policy; see [the docs](https://cert-manager.io/docs/projects/approver-policy/)
for details on how to install properly. Alternatively, the instructions below should work!

Note that trust-manager's Helm chart includes a Certificate which you'll need to approve manually.

Installation instructions are given below as a guide, using either [kind](https://kind.sigs.k8s.io/) or whatever cluster you have configured.

```console
# if you already have a cluster, skip this step
kind create cluster --name ap

# install cert-manager with approver disabled
helm upgrade -i -n cert-manager cert-manager jetstack/cert-manager --set extraArgs={--controllers='*\,-certificaterequests-approver'} --set installCRDs=true --create-namespace

# install approver-policy
helm upgrade -i -n cert-manager cert-manager-approver-policy jetstack/cert-manager-approver-policy --wait

# install trust-manager; this might appear to hang because it'll be waiting for the certificate to be issued.
# if it hangs, use the command below to approve the CertificateRequest!
helm upgrade -i -n cert-manager trust-manager jetstack/trust-manager --wait

# approve trust-manager CertificateRequest (this actually approves all certs in the cert-manager namespace; you might want to be more careful in prod)
kubectl get -n cert-manager cr -o go-template="{{range .items}}{{printf \"%s\n\" .metadata.name}}{{end}}" | xargs -I% cmctl approve -n cert-manager %
```

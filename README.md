# Image verification with Sysdig through the image verification
This PoC covers image verification and events in Sysdig Secure.
## Prerequisites

AWS account with permissions to manage EKS and EC2 modules
DockerHub account configured
Docker Container Runtime installed and running
Sysdig UI

### Image signature and verification
Install cosign with homebrew
```
$ brew install cosign
```

Clone the repository
```
$ git clone git@github.com:gmontalvoy/ws_sysdig.git
$ cd ws_sysdig
```

Generate key-pairs
```
$ cosign generate-key-pair
Enter password for private key: 
Enter password for private key again: 
Private key written to cosign.key
Public key written to cosign.pub
```

The above create the key to sign the image and a public key used by [Kyverno](https://kyverno.io) 

Use the provided <em>Dockerfile</em> in this repository to create the image
```
$ docker build -t DOCKERUSER/test:signed -f Dockerfile_signed
$ docker push DOCKERUSER/test:signed

$ docker build -t DOCKERUSER/test:unsigned -f Dockerfile_unsigned
$ docker push DOCKERUSER/test:unsigned
```

Sign one of the images
```
$ cosign sign --key <KEYNAME>.key DOCKERUSER/test:signed
```

A new tag will appear on your repository which is the signature for the specific image, signature used to verify the image

```
$ cosign verify --key <KEYNAME>.pub DOCKERUSER/test:signed

Verification for index.docker.io/gmontalvoy/test:signed --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - Existence of the claims in the transparency log was verified offline
  - The signatures were verified against the specified public key
```

## Deploy cluster and install Kyverno

Kyverno is a Kubernetes Native Policy Management which, through an admission controller and a set of objects manage and controls the policy allowing, denying and informing the platform administrator about policy violations or warnings.

This PoC uses Kyverno to set a policy that only allow signature-verified images on production namespaces while allowing unsigned images on any other non-production namespace

Deploy cluster with <em>eksctl</em> or use your own one
```
eksctl create cluster \
    --version 1.24 \
    --name test123 \
    --node-type a1.large \
    --nodes 3 \ 
    --region us-east-2

```

For this PoC, there will be two namespaces <em>prod/dev</em>, kustomize will be taken care of this

``` 
$ kubectl create -k dev/
namespace/dev created
$ kubectl create -k prod/
namespace/prod created
```


[Kyverno](https://kyverno.io) can be deployed with Helm as described in [Kyverno Docs](https://kyverno.io/docs/installation/#install-kyverno-using-helm)

```
helm install kyverno kyverno/kyverno -n kyverno --create-namespace --set replicaCount=3
```
Kyverno policies can be used to manage certain situations before the admission controller allow the object to be created.
Let's create the policies

```
$ kubectl create -f cpolicy.yaml
clusterpolicy.kyverno.io/check-image-signature created
```

The above policy will prevent any unsigned image on a "prod" namespace while allowing deployment on any other namespace.

Kyverno policies are created as <em>clusterPolicy</em> objects

```
$ kubectl get clusterpolicy
NAME                    BACKGROUND   VALIDATE ACTION   READY   AGE
check-image-signature   false        Audit             true    24h

```

Let's deploy an unsigned image to prod namespace
```
kc create deployment unsigned --image=gmontalvoy/test:unsigned -n prod
error: failed to create deployment: admission webhook "mutate.kyverno.svc-fail" denied the request: 

policy Deployment/prod/unsigned for resource violation: 

check-image-signature:
  autogen-verify-image: |
    failed to verify image docker.io/gmontalvoy/test:unsigned: .attestors[0].entries[0].keys: no matching signatures:

```

If deployed on non-prod namespace a warning about policy validation shows and an event is created
```
❯ kc create deployment unsigned --image=gmontalvoy/test:unsigned -n dev
Warning: policy check-image-signature.autogen-verify-image: unverified image docker.io/gmontalvoy/test@sha256:6234a29e5fe3235109059262e86d4ffddcc5785ce5abbf73973aa25cc9fb3915
deployment.apps/unsigned created
```

Policy results can be obtained through the inspection of <em>policyreport</em> or <em>polr</em>.

```
  results:
  - message: unverified image docker.io/gmontalvoy/test@sha256:6234a29e5fe3235109059262e86d4ffddcc5785ce5abbf73973aa25cc9fb3915
    policy: check-image-signature
    resources:
    - apiVersion: v1
      kind: Pod
      name: unsigned-85755b54c-flw7x
      namespace: dev
      uid: a8e4f91e-cbea-4fb5-9bb4-53e0cbd9f28c
    result: fail
    rule: verify-image
    scored: true
    severity: medium
    source: kyverno
    timestamp:
      nanos: 0
      seconds: 1679830249
```

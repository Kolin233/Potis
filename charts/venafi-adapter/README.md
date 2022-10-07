# Nirmata Venafi Adapter

[Kyverno](https://kyverno.io) is a Kubernetes Native Policy Management engine. The Nirmata Venafi Adapter helps extract public keys, certificates or certchains stored securely in [Venafi CodeSign Protect](https://www.venafi.com/platform/codesign-protect). These are stored in configmap fields which in-turn can be referred to from Kyverno image-verify policies.

## Introduction

This chart bootstraps a Venafi Adapter (a.k.a. ImageKey controller) on a [Kubernetes](http://kubernetes.io) cluster with [Kyverno](https://kyverno.io) installed, using the [Helm](https://helm.sh) package manager.

## Installation

```
# 1. Add Kyverno Helm Repository

helm repo add nirmata https://nirmata.github.io/kyverno-charts/

# 2. Install venafi-adapter from nirmata helm repo in the nirmata-kyverno-monitor namespace, with desired parameters.

helm install venafi-adapter nirmata/venafi-adapter --namespace nirmata-venafi-adapter --create-namespace --set imagePullSecret.username=someuser,imagePullSecret.password=somepassword

# 3. Check pods are running
kubectl -n <namespace> get pods 

# 4. Check CRD is created
kubectl -n <namespace> get ImageKey
```

## Test a sample policy
For a sample use-case, create an imagekey CR, verify that it fetches venafi certs or keys and saves to a configmap. And then create a policy to verify that it is accepted/blocked as expected.
```

# 1 Create a password secret for venafi environment
kubectl create secret generic venafi-pwd-secret -n nirmata-venafi-adapter --from-literal password=<your-password> --as system:serviceaccount:<namespace>:imagekey-controller

# 2. If needed, create an additional secret to convey to Venafi that an additional X.509 cert has to be trusted. Needed for some corner cases when enterprise customers use internal certificates. E.g. 
kubectl create secret generic venafi-addl-cert-secret -n nirmata-venafi-adapter --from-file <addl-cert-key-filename>

# 3. Create the CR yaml imagekey.yaml. E.g. 

apiVersion: security.nirmata.io/v1
kind: ImageKey
metadata:
  name: imagekey1
  namespace: nirmata-venafi-adapter
spec:
  venafiPKCS11Config:
    authURL: "https://yourorg-tpp.se.venafi.com/vedauth"
    hsmURL: "https://yourorg-tpp.se.venafi.com/vedhsm"
    username: your-venafi-username
    passwordSecretName: <namespace>/<venafi-env1-password-secret>/<key-in-secret>
    label: <Environment-Label>
    interval: <in minutes>
    hostAlias: NA
    configMap: <namespace>/<some-configmap>/<confmap key matching that in policy>
    fetchType: <pubkey, certificate or certchain>
    imagepullsecret: <secretname for private registries>
    keyfetchClientRegistry: <"" or prefix before nirmata/ in uri, e.g. ghcr.io>
    additionalCertSecretName: <"" or e.g. default/addl-venafi-cert-secret/addl-cert-secret-key>

# 4. Apply the CR
kubectl -n <namespace> create imagekey.yaml

# 5. Ensure that the first job runs and downloads the specified key to configmap specified
kubectl -n <namespace> get cm <config map name in CR> -o yaml

# 6. Create a Kyverno imageverify policy referring to the configmap field. [See this](https://kyverno.io/policies/other/verify_image/)

# 7. Create pods and check that they are blocked or allowed based on whether they are signed using Venafi keys or not. 
k run venafisignedpod --image=ghcr.io/some-user/some-image:signed-by-me   
```

## Uninstalling the Chart

To uninstall/delete the `venafi-adapter` deployment:

```console
helm -n <namespace> uninstall venafi-monitor
```

The command removes all the Kubernetes components associated with the chart and deletes the release. 

Note: The crd `imagekeys.security.nirmata.io` needs to be deleted manually

## Configuration

The following table lists the configurable parameters of the kyverno chart and their default values.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| namespace | string | `nirmata-kyverno-monitor` | Namespace to install kyverno-monitor resources |
| imagePullSecret.name | string | `image-pull-secret` | Imagepull secret name that will store private image registry info |
| imagePullSecret.create | boolean | `true` | Whether to create the image pullsecret. Need to specify the secret name, username, password |
| imagePullSecret.username | string |  | Private registry username if secret is to be created |
| imagePullSecret.password | string |  | Private registry password if secret is to be created |
| venafiAdapterImage | string | `ghcr.io/nirmata/imagekey-controller` | Venafi adapter image |
| venafiAdapterImageTag | string | `0.1.0` | Venafi adapter image tag. If empty, appVersion in Chart.yaml is used |

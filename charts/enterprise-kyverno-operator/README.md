# Helm Chart for Enterprise Kyverno
Enterprise Kyverno is a Kubernetes Operator to manage lifecycle of Kyverno, Adapters and Nirmata supported policies. 

## Prerequisites
### Get license key
You need a license key to run Enterprise Kyverno. If you are using Nirmata Enterprise for Kyverno, it is available in the UI. Else contact support@nirmata.com.

### (Optional) Install cert-manager
Kyverno Operator uses webhooks to provide enhanced functionality such as logging user information in resource change events logged into the Kubernetes event stream, and some enhanced semantic checks for custom resources. By default, Kyverno Operator does not install webhooks. 

But if webhooks are to be enabled for this enhanced functionality, then it needs an installation of `cert-manager` for webhook certificate management. Install cert-manager by following instructions [here](https://cert-manager.io/docs/installation/). Typically,
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/cert-manager.yaml
```

### (Optional) If a custom CA is used, create a configmap corresponding to the same with key custom-ca.pem. E.g.
Create the namespace
```bash
kubectl create namespace enterprise-kyverno-operator
```
Create configmap in the namespace
```bash
kubectl -n nirmata-kyverno-operator create configmap <e.g. ca-store-cm> --from-file=custom-ca.pem=<cert file e.g. some-cert.pem>
```

## Getting Started
Add the chart repository and install the chart
```bash
helm repo add nirmata https://nirmata.github.io/kyverno-charts
helm repo update nirmata

helm install enterprise-kyverno-operator nirmata/enterprise-kyverno-operator -n enterprise-kyverno-operator --create-namespace --set licenseKey=<licenseKey>[,apiKey=<api key>]
```
Helm Chart parameters for further fine-tuning the above helm install are described in the [Helm Chart Values](#helm-chart-values) section below.

Additional parameters corresponding to custom CA or HTTP proxies, NO_PROXY should be provided as needed. E.g.
```bash
--set customCAConfigMap=<e.g. ca-store-cm> --set systemCertPath=<e.g. /etc/ssl/certs>  --set "extraEnvVars[0].name=HTTP_PROXY" --set "extraEnvVars[0].value=<e.g. http://test.com:8080>" ...
```

View various Resources created
```bash
kubectl -n enterprise-kyverno-operator get kyvernoes.security.nirmata.io #(CR that defines kyverno settings)
kubectl -n enterprise-kyverno-operator get policysets.security.nirmata.io #(CRs corresponding to default policysets installed)

kubectl -n kyverno get po #(should show Kyverno pods getting ready)
kubectl get cpol #(should show policies installed by initial policysets)
```

If you need to modify Kyverno configuration, change CR directly or via Helm Upgrade
```bash
kubectl -n enterprise-kyverno-operator edit kyvernoes.security.nirmata.io kyverno (and set replicas to 3)

helm upgrade enterprise-kyverno-operator nirmata/enterprise-kyverno-operator -n enterprise-kyverno-operator --create-namespace --set licenseKey=<licenseKey> --set kyverno.replicas=3
```

Removing a Policy Set CR removes policies contained in it
```bash
kubectl -n enterprise-kyverno-operator delete policysets best-practices
```

To remove Enterprise Kyverno and components
```bash
helm uninstall -n enterprise-kyverno-operator enterprise-kyverno-operator
```

## Configure Adapters
Note: CIS Adapter is currently available within Kyverno Operator as a Release Candidate only. Use the `--devel` flag in helm commands to exercise the CIS Adapter feature.

Adapters such as AWS, CIS and others can be configured by setting appropriate flags corresponding to that adapter. In general, we need to provide 2 flags
- Flags to create RBAC resources such as clusterroles, bindings, serviceaccounts. E.g. `--set awsAdapter.rbac.create=true` or `--set cisAdapter.serviceAccount.create=true`
- Flags to create the CR for that adapter at chart install time itself. E.g. `--set awsAdapter.createCR=true`

See the Helm chart values below for specifics.


## Helm Chart Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| nameOverride | string | `nil` | Override the name of the chart |
| fullnameOverride | string | `nil` | Override the expanded name of the chart |
| enableWebhook | bool | `false` | Enable operator webhooks for enhanced error checks and user info in audit log |
| licenseKey | string | `nil`| License key (required) |
| apiKey | string | `nil` | License server API key |
| rbac.create | bool | `true` | Enable RBAC resources creation |
| rbac.operatorHasAdminPerms | bool | `false` | Whether operator has admin permissions to install CRD and RBAC |
| rbac.serviceAccount.name | string | `nil` | Service account name when `rbac.create` is set to `false` |
| image.repository | string | `"ghcr.io/nirmata/enterprise-kyverno-operator"` | Image repository |
| image.pullPolicy | string | `"IfNotPresent"` | Image pull policy |
| image.tag | string | `nil` | Image tag (defaults to chart app version) |
| image.imagePullSecrets.registry | string | `ghcr.io` | Image pull secret registry |
| image.imagePullSecrets.name | string | `image-pull-secret` | Image pull secret name |
| image.imagePullSecrets.create | bool | `false` | Whether to create image pull secret |
| image.imagePullSecrets.username | string | `nil` | Username for image pull secret registry |
| image.imagePullSecrets.password | string | `nil` | Password for image pull secret registry |
| kyverno.createCR | bool | `true` | Create a CR that describes Kyverno to be managed by operator |
| kyverno.replicaCount | int | `1` | Kyverno replicas |
| kyverno.rbac.create | bool | `true` | Enable Kyverno RBAC resources creation |
| kyverno.rbac.serviceAccount.create | string | `nil` | Whether to create Kyverno service account |
| kyverno.rbac.serviceAccount.clusterRole.extraResources | list | `[]` | Extra resource permissions to add to the Kyverno cluster role |
| kyverno.generatecontrollerExtraResources | list | `[]]` | Additional resources to be added to kyverno controller RBAC permissions |
| kyverno.image.repository | string | `"ghcr.io/nirmata/kyverno"` | Kyverno Image repository |
| kyverno.image.pullPolicy | string | `"IfNotPresent"` | Kyverno Image pull policy |
| kyverno.image.tag | string | `v1.9.1-n4k.nirmata.1` | Image tag (defaults to chart app version) |
| kyverno.enablePolicyExceptions| bool | `false` | Enable policyexceptions feature in Kyverno 1.9+ |
| kyverno.helm | object | `helm.rbac.serviceAccount.name=kyverno` | Free form yaml section with helm parameters in Kyverno chart. See all parameters [here](https://github.com/nirmata/kyverno-charts/tree/main/charts/nirmata#values) |
| policies.policySets | list | `["best-practices", "pod-security"]` | Default policy sets to be installed along with operator |
| awsAdapter.rbac.create | bool | false | Create RBAC resources for Kyverno AWS Adapter, if AWS Adapter is going to be enabled now (through the awsAdapter.createCR helm param below) or later |
| awsAdapter.createCR | bool | false | Enable AWS Adapter by creating its Adapter Config CR |
| awsAdapter.eksCluster.name | string | `nil` | EKS Cluster name. Required if awsAdapter.createCR is true |
| awsAdapter.eksCluster.region | string | `nil` | EKS Cluster region. Required if awsAdapter.createCR is true |
| awsAdapter.eksCluster.roleARN | string | `nil` | EKS Cluster roleARN. Required if awsAdapter.createCR is true |
| awsAdapter.helm | object | `nil` | Free form yaml section with helm parameters in Kyverno AWS Adapter Helm chart. Needed only if awsAdapter.createCR is true. See all parameters [here](https://github.com/nirmata/kyverno-aws-adapter/tree/main/charts/kyverno-aws-adapter#values) |
| cisAdapter.rbac.create | bool | false | Create RBAC resources for CIS Adapter, if CIS Adapter is going to be enabled now (through the cisAdapter.createCR param below) or later |
| cisAdapter.serviceAccount.create | bool | false | Create Service Account for CIS Adapter, if CIS Adapter is going to be enabled now (through the cisAdapter.createCR param below) or later |
| cisAdapter.createCR | bool | false | Enable CIS Adapter by creating its Adapter Config CR |
| cisAdapter.helm | object | `nil` | Free form yaml section with helm parameters in CIS Adapter Helm chart. Needed only if cisAdapter.createCR is true. See all parameters [here](https://github.com/nirmata/kyverno-charts/tree/main/charts/kube-bench-adapter#values) |

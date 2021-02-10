# AWS Pod Identity Webhook

This chart will install the [Amazon EKS Pod Identity Webhook](https://github.com/aws/amazon-eks-pod-identity-webhook). This tool allows you to specify IAM Roles for Kubernetes Service Accounts. This allows a pod to assume a IAM role.

Further details can be found here: https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts-technical-overview.html

This chart can generate certificate using [helm genCA](https://helm.sh/docs/chart_template_guide/function_list/#cryptographic-and-security-functions)

OR

It may ask the webhook server to generate one with [CSR Kubernetes feature](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/)

## Prerequisites

- Kubernetes 1.12+

For those who are not using EKS, please take a look at https://registry.terraform.io/modules/int128/kubernetes-irsa/aws/latest

## Installing the Chart

You first need to retrieve `ca.crt` from your cluster as this is used as a value for the chart:

```sh
secret_name=$(kubectl get sa default -o jsonpath='{.secrets[0].name}')
export CA_BUNDLE=$(kubectl get secret/$secret_name -o jsonpath='{.data.ca\.crt}' | tr -d '\n')
```

Then install the chart:

:warning: The first time, deploy `pod-identity-webhook` with only 1 replica to avoid creating as much CSR as replica count whereas only 1 CSR approval is required.

```sh
helm upgrade -i pod-identity-webhook eks/aws-pod-identity-webhook \
--namespace kube-system --set caBundle="${CA_BUNDLE}" --set replicas=1
```

After installation you need to approve the certificate. Follow the chart notes after installation for this step.

The webhook will request a new CSR prior to expiration. This new CSR will also need to be manually approved.

When the value `prometheus-operator.enable_alerting_rule` is `true`, an alert is triggered when a CSR approvment is pending.

Then approve the certificate:

```bash
kubectl certificate approve $(kubectl get csr -o jsonpath='{.items[?(@.spec.username=="system:serviceaccount:kube-system:pod-identity-webhook")].metadata.name}')
```

### Check the generated certificate (optional)

After this step, a secret named after the value `.tlsSecretName` (default `pod-identity-webhook`) must have been created.

```bash
kubectl get secret -n kube-system pod-identity-webhook -o json | jq -r '.data."tls.crt"' | base64 -d | openssl x509  -text -noout

Certificate:
    Data:
        Version: 3 (0x2)
...
        Issuer: CN = kubernetes
        Validity
            Not Before: Jul 31 10:09:00 2020 GMT
            Not After : Jul 31 10:09:00 2021 GMT
        Subject: CN = pod-identity-webhook.kube-system.svc
...
```

For macos user,

```bash
kubectl get secret -n kube-system pod-identity-webhook -o json | jq -r '.data."tls.crt"' | base64 -D | openssl x509  -text -noout
```

> :warning: See the `Not After` date, This certificate has a one year validity. If you enable values `prometheus_operator.enable_service_monitor` and `prometheus-operator.enable_alerting_rule`, a Prometheus alert rule is set.

TODO: How to renew this certificate ? Is there a new certificate emit and which must be approved ?

## Uninstalling the Chart

To delete the chart:

```sh
helm delete --purge pod-identity-webhook
kubectl delete secret -n kube-system pod-identity-webhook # `pod-identity-webhook` must be the value if `.tlsSecretName`
# Delete the CSR
kubectl delete csr -o jsonpath="{.items[?(@.spec.username=='system:serviceaccount:${HELM_NAMESPACE}:pod-identity-webhook')].metadata.name}"
```

## Configuration

The following table lists the configurable parameters for this chart and their default values.

| Parameter                   | Description                           | Default                                                                 |
| ----------------------------|---------------------------------------|-------------------------------------------------------------------------|
| `tlsSecretName`             | Name of the secret containing the     | `pod-identity-webhook`                                                  |
| `annotationPrefix`          | Prefix for annotation                 | `eks.amazonaws.com`                                                     |
| `tokenAudience`             | Token audience                        | `sts.amazonaws.com`                                                     |
| `caBundle`                  | CA cert bundle data                   | None. Must be provided on chart install                                 |
| `image.repository`          | Image repository                      | `xxxxxxxxxxxxx.dkr.ecr.us-west-2.amazonaws.com/eks/pod-identity-webhook` |
| `image.tag`                 | Image tag                             | `latest`                                                                |
| `image.pullPolicy`          | Container pull policy                 | `IfNotPresent`                                                          |
| `replicas`                  | Number of deployment replicas         | `3`                                                                     |
| `fullnameOverride`          | Override the fullname of the chart    | `nil`                                                                   |
| `nameOverride`              | Override the name of the chart        | `nil`                                                                   |
| `priorityClassName`         | Set a priority class for pods         | `nil`                                                                   |
| `resources.requests.cpu`    | pod CPU request                       | `100m`                                                                  |
| `resources.requests.memory` | pod memory request                    | `64Mi`                                                                  |
| `resources.limits.cpu`      | pod CPU limit                         | `2000m`                                                                 |
| `resources.limits.memory`   | pod memory limit                      | `1Gi`                                                                   |
| `nodeSelector`              | Node labels for pod assignment        | `{}`                                                                    |
| `tolerations`               | Optional deployment tolerations       | `[]`                                                                    |
| `affinity`                  | Map of node/pod affinities            | `{}`                                                                    |
| `prometheus_operator.enable_alerting_rule`                  | Deploy Promtheus Operator Alerting rule            | `true`                                                                    |
| `prometheus_operator.enable_service_monitor`                  | Enable metrics scraping with via ServiceMonitor            | `{}`                                                                    |
| `generateAdmissionControllerCerts` | Auto-generate TLS certificates for admission controller. | `false` |
| `admissionControllerCert` | Manually set admission controller certificate. | Unset |
| `admissionControllerKey` | Manually set admission controller key. | Unset |

Specify each parameter using the `--set key=value[,key=value]` argument to `helm install` or provide a YAML file containing the values for the above parameters:

```sh
helm update -i pod-identity-webhook eks/aws-pod-identity-webhook --namespace kube-system --values values.yaml
```

## Note about certs

There is 3 solutions supported by this chart.

### 1/

To use Kubernetes CA, you just need to set `generateAdmissionControllerCerts` to `false`. And then, at initial installation, you need to approve the CSR as describe above.

### 2/

For a fully automated certificate generation (certs expires in 10 years with this method):
`generateAdmissionControllerCerts` must be `true`. Then `caBundle`, `admissionControllerCert`, `admissionControllerKey` will be populated automatically.

### 3/

For manual certificate, `generateAdmissionControllerCerts` must be `false` and then you must provide `caBundle`, `admissionControllerCert`, `admissionControllerKey`.

## Source

This chart has been created for eks-chart official Helm repository and for CNCF Helm official repo, which never got published.

https://github.com/helm/charts/pull/17099
https://github.com/aws/eks-charts/pull/28

we have adapted it to our needs.

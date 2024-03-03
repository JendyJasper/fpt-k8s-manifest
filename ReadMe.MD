#### Setup kubeconfig
`aws eks --region us-east-1 update-kubeconfig --name fpt-cluster`

        |---------------------------------------------------|
        |                                                   |
        |---------------------------------------------------|

#### Install the custom helm chart for the fpt python-reddis app
`helm upgrade -i --reuse-values fpt-redis-app custom_charts/fpt-flask-redis/. -f custom_charts/fpt-flask-redis/fpt_flask_app_values.yml`

        |---------------------------------------------------|
        |                                                   |
        |---------------------------------------------------|

## Cert Manager installation

#### cert-manager CustomResourceDefinition resources
`kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.3/cert-manager.crds.yaml`

#### Add the Jetstack Helm repository
`helm repo add jetstack https://charts.jetstack.io`

#### Install the cert-manager helm chart
`helm upgrade -i  cert-manager --namespace cert-manager --version v1.13.3 jetstack/cert-manager -f cert_manager_values.yml`

        |---------------------------------------------------|
        |                                                   |
        |---------------------------------------------------|


## AWS Load Balancer Controller installations

#### Create IAM OIDC provider
`eksctl utils associate-iam-oidc-provider \
    --region <aws-region> \
    --cluster <your-cluster-name> \
    --approve`

#### Download IAM policy for the AWS Load Balancer Controller
`curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json`

#### Create an IAM policy called AWSLoadBalancerControllerIAMPolicy. Take note of the policy ARN that is returned
`aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam-policy.json`

#### Create a IAM role and ServiceAccount for the Load Balancer controller, use the ARN from the step above
`eksctl create iamserviceaccount \
--cluster=<cluster-name> \
--namespace=kube-system \
--name=aws-load-balancer-controller \
--attach-policy-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
--approve`

#### Install the TargetGroupBinding CRDs:
`kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"`

## install alb controller
`helm upgrade -i aws-load-balancer-controller aws/aws-load-balancer-controller -n ingress-controller -f alb_controller_values.yml`

> if service account already exists and pods refuse to come up for aws load balancer ##controller, delete the service account and create again

`eksctl delete iamserviceaccount --name aws-load-balancer-controller --namespace ingress-controller --cluster fpt-cluster`


        |---------------------------------------------------|
        |                                                   |
        |---------------------------------------------------|

## EFS CSI Driver Installation

#### Add the Helm repo.
`helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/`

#### Update the repo.
`helm repo update aws-efs-csi-driver`

> Install a release of the driver using the Helm chart. Use the complete command if. If you already created a service account by following Create an IAM policy and role for Amazon EKS, then add the following arguments. and if If you don't have outbound access to the Internet, add the following arguments too To force the Amazon EFS CSI driver to use FIPS for mounting the file system, add the following argument. If running on fargate, scheduling on kube-system is a challenge i'm yet to resolve

`helm upgrade --install efs-driver aws-efs-csi-driver/aws-efs-csi-driver --namespace default --set controller.serviceAccount.create=false --set controller.serviceAccount.name=efs-csi-controller-sa --set sidecars.livenessProbe.image.repository=602401143452.dkr.ecr.region-code.amazonaws.com/eks/livenessprobe --set sidecars.node-driver-registrar.image.repository=602401143452.dkr.ecr.region-code.amazonaws.com/eks/csi-node-driver-registrar --set sidecars.csiProvisioner.image.repository=602401143452.dkr.ecr.region-code.amazonaws.com/eks/csi-provisioner --set useFips=true`


`eksctl create iamserviceaccount \
--cluster=fpt-cluster \
--namespace=ingress-controller \
--name=aws-load-balancer-controller \
--attach-policy-arn=arn:aws:iam::571207880192:policy/AWSLoadBalancerControllerIAMPolicy \
--approve`




        |---------------------------------------------------|
        |                                                   |
        |---------------------------------------------------|

### Prometheus Stack

#### Add repository
`helm repo add prometheus-community https://prometheus-community.github.io/helm-charts`

#### Update repo
`helm repo update`

#### Install Helm Chart
`helm upgrade -i prometheus-stack prometheus-community/kube-prometheus-stack -f prometheus_stack_values.yml --reuse-values -n monitoring`

### Persistence volumes
> the additional iam service account to create for prometheus and grafana is prometheus-stack-grafana. make sure the nodes security group or cidr of the subnets or vpc has access to the efs whether using fargate or ec2 add both efs and ebs policy to the fargate nodes / ec2 nodes as additional nodes. add the iam service account roles to the efs_csi and ebs_csi add on the irsa should have the policies  of efs and ebs

#### prometheus stack additional notes
- create the namespace of the prometheus-stack
- create the storage class
- create the service account for the ebs or efs
- if using static provisioning, create the persistence volume too


### Sealed secret
- `helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets`
- `helm install kube-secret sealed-secrets/sealed-secrets`

> Install the client-side tool (kubeseal) as explained in the docs below: https://github.com/bitnami-labs/sealed-secrets#installation-from-source

- KUBESEAL_VERSION=''  Set this to, for example, KUBESEAL_VERSION='0.23.0'
- https://api.github.com/repos/bitnami-labs/sealed-secrets/tags  to get the latest version
- Then run the below command

```
wget "https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION:?}/kubeseal-${KUBESEAL_VERSION:?}-linux-amd64.tar.gz"
tar -xvzf kubeseal-${KUBESEAL_VERSION:?}-linux-amd64.tar.gz kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```

## Seal the secret
- `kubeseal --controller-name NAME_OF_SEALED_SECRET_SERVICE --controller-namespace NAMESPACE_SEALED_SECRET_IS_RUNNING --format yaml < ORIGINAL_SECRET_FILE > GENERATED_SEALED_SECRET_NAME`
- `kubeseal --controller-name kube-secret-sealed-secrets --controller-namespace default --format yaml < infr
astructure/cert_manager/aws_secret.yml > infrastructure/cert_manager/aws_sealed_secret.yml`


## External secret operator
- add the repo: `helm repo add external-secrets https://charts.external-secrets.io`
- install: `helm install external-secrets \
   external-secrets/external-secrets \
    -n external-secrets \
    --create-namespace `

### create a secret store
```
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secretstore
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        secretRef:
          accessKeyIDSecretRef:
            name: aws-secret
            key: access-key-id
          secretAccessKeySecretRef:
            name: aws-secret
            key: secret-access-key
```

> when encoding, `echo "" | base64` adds a newline. Use instead: `echo -n "" | base64`


## Velero activation for cluster back up and restore
- Create S3 bucket
- Create the IAM user
- Attach policies to give velero the necessary permissions:
```
cat > velero-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeVolumes",
                "ec2:DescribeSnapshots",
                "ec2:CreateTags",
                "ec2:CreateVolume",
                "ec2:CreateSnapshot",
                "ec2:DeleteSnapshot"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:DeleteObject",
                "s3:PutObject",
                "s3:AbortMultipartUpload",
                "s3:ListMultipartUploadParts"
            ],
            "Resource": [
                "arn:aws:s3:::${BUCKET}/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::${BUCKET}"
            ]
        }
    ]
}
EOF
```
- attach the policy to the user
- Create an access key for the user
- Create a Velero-specific credentials file (credentials-velero) in your local directory:
```
[default]
aws_access_key_id=<AWS_ACCESS_KEY_ID>
aws_secret_access_key=<AWS_SECRET_ACCESS_KEY>
```
> The access key ID and Secret access key have to be those of the newly created iam user

#### Install and start Velero
> useful links for the process https://dev.to/mxglt/how-to-setup-a-dr-for-your-k8s-cluster-with-velero-1l77
- see https://github.com/vmware-tanzu/velero/releases for latest cli release of velero
- download the cli
```
# Download the Velero release
curl -L -o /tmp/velero.tar.gz curl -L -o /tmp/velero.tar.gz https://github.com/vmware-tanzu/velero/releases/download/v1.13.0/velero-v1.13.0-linux-amd64.tar.gz


# Unzip it
tar -C /tmp -xvf /tmp/velero.tar.gz

# Move to the local/bin folder
mv /tmp/velero-v1.13.0-linux-amd64/velero /usr/local/bin/velero
chmod +x /usr/local/bin/velero

# Test the velero command, and it should work!
velero --help
```
#### install velero to the cluster
> useful link for the process https://github.com/vmware-tanzu/velero-plugin-for-aws?tab=readme-ov-file#Install-and-start-Velero
- see https://github.com/vmware-tanzu/velero-plugin-for-aws/releases for velero aws plugin latest release and adjust accordingly

```
velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.9.0 \
    --bucket $BUCKET \
    --backup-location-config region=$REGION \
    --snapshot-location-config region=$REGION \
    --secret-file ./credentials-velero
```
> backup eveything in the cluster
- velero backup create <backup-name> --include-namespaces="*" --snapshot-volumes --wait
- velero backup create fpt-backup-1 --include-namespaces="*" --snapshot-volumes --wait

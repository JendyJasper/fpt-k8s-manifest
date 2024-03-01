#Setup kubeconfig
`aws eks --region us-east-1 update-kubeconfig --name fpt-cluster`

        |---------------------------------------------------|
        |                                                   |
        |---------------------------------------------------|

#Install the custom helm chart for the fpt python-reddis app
`helm upgrade -i --reuse-values fpt-redis-app custom_charts/fpt-flask-redis/. -f custom_charts/fpt-flask-redis/fpt_flask_app_values.yml`

        |---------------------------------------------------|
        |                                                   |
        |---------------------------------------------------|

#Cert Manager installation

##cert-manager CustomResourceDefinition resources
`kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.3/cert-manager.crds.yaml`

## Add the Jetstack Helm repository
`helm repo add jetstack https://charts.jetstack.io`

## Install the cert-manager helm chart
`helm upgrade -i  cert-manager --namespace cert-manager --version v1.13.3 jetstack/cert-manager -f cert_manager_values.yml`

        |---------------------------------------------------|
        |                                                   |
        |---------------------------------------------------|


#AWS Load Balancer Controller installations

##Create IAM OIDC provider
`eksctl utils associate-iam-oidc-provider \
    --region <aws-region> \
    --cluster <your-cluster-name> \
    --approve`

##Download IAM policy for the AWS Load Balancer Controller
`curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json`

##Create an IAM policy called AWSLoadBalancerControllerIAMPolicy. Take note of the policy ARN that is returned
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam-policy.json

##Create a IAM role and ServiceAccount for the Load Balancer controller, use the ARN from the step above
eksctl create iamserviceaccount \
--cluster=<cluster-name> \
--namespace=kube-system \
--name=aws-load-balancer-controller \
--attach-policy-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
--approve

##Install the TargetGroupBinding CRDs:
kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"

##install alb controller
helm upgrade -i aws-load-balancer-controller aws/aws-load-balancer-controller -n ingress-controller -f alb_controller_values.yml

##if service account already exists and pods refuse to come up for aws load balancer ##controller, delete the service account and create again
eksctl delete iamserviceaccount --name aws-load-balancer-controller --namespace ingress-controller --cluster fpt-cluster


        |---------------------------------------------------|
        |                                                   |
        |---------------------------------------------------|

#EFS CSI Driver Installation

##Add the Helm repo.
helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/

##Update the repo.
helm repo update aws-efs-csi-driver

##Install a release of the driver using the Helm chart. Use the complete command if
## If you already created a service account by following Create an IAM policy and role for Amazon EKS, then add the following arguments.
## and if If you don't have outbound access to the Internet, add the following arguments too
## To force the Amazon EFS CSI driver to use FIPS for mounting the file system, add the following argument. If running on fargate, scheduling on kube-system is a challenge i'm yet to resolve
helm upgrade --install efs-driver aws-efs-csi-driver/aws-efs-csi-driver --namespace default --set controller.serviceAccount.create=false --set controller.serviceAccount.name=efs-csi-controller-sa --set sidecars.livenessProbe.image.repository=602401143452.dkr.ecr.region-code.amazonaws.com/eks/livenessprobe --set sidecars.node-driver-registrar.image.repository=602401143452.dkr.ecr.region-code.amazonaws.com/eks/csi-node-driver-registrar --set sidecars.csiProvisioner.image.repository=602401143452.dkr.ecr.region-code.amazonaws.com/eks/csi-provisioner --set useFips=true


eksctl create iamserviceaccount \
--cluster=fpt-cluster \
--namespace=ingress-controller \
--name=aws-load-balancer-controller \
--attach-policy-arn=arn:aws:iam::571207880192:policy/AWSLoadBalancerControllerIAMPolicy \
--approve




        |---------------------------------------------------|
        |                                                   |
        |---------------------------------------------------|

#Prometheus Stack

##Add repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

##Update repo
helm repo update

##Install Helm Chart
helm upgrade -i prometheus-stack prometheus-community/kube-prometheus-stack -f prometheus_stack_values.yml --reuse-values -n monitoring

#Persistence volumes
the additional iam service account to create for prometheus and grafana is prometheus-stack-grafana
make sure the nodes security group or cidr of the subnets or vpc has access to the efs whether using fargate or ec2
add both efs and ebs policy to the fargate nodes / ec2 nodes as additional nodes
add the iam service account roles to the efs_csi and ebs_csi add on
the irsa should have the policies  of efs and ebs

#prometheus stack additional notes
create the namespace of the prometheus-stack
create the storage class
create the service account for the ebs or efs
if using static provisioning, create the persistence volume too
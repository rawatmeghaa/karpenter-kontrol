Karpenter : 
Karpenter only provision nodes not pods .It launhes node when pod is not scheduled on existing node due to memory or taint nad toleration
Have the ability to launch the correct size of instance type , bcz they have nodepool for whic it select the correct conatiner mainly spot and code optimize

Karpenter doc: https://karpenter.sh/docs/getting-started/getting-started-with-karpenter/
cloudformation.yaml : https://karpenter.sh/docs/reference/cloudformation/

1.Install utilities
     awscli , kubectl , eksctl , helm

2.Set environment variables

 
export KARPENTER_NAMESPACE="karpenter"
export KARPENTER_VERSION="1.6.0"
export CLUSTER_NAME="cluster_name"




echo "${KARPENTER_NAMESPACE}" "${KARPENTER_VERSION}" "${K8S_VERSION}" "${CLUSTER_NAME}" "${AWS_DEFAULT_REGION}" "${AWS_ACCOUNT_ID}" "${TEMPOUT}" "${ALIAS_VERSION}"


Ami : 
aws ssm get-parameter --name "/aws/service/eks/optimized-ami/${K8S_VERSION}/amazon-linux-2023/x86_64/standard/recommended/image_id" --query Parameter.Value | xargs aws ec2 describe-images --query 'Images[0].Name' --image-ids


export AWS_PARTITION="aws" # if you are not using standard partitions, you may need to configure to aws-cn / aws-us-gov
 export CLUSTER_NAME="MY_CLUSTER"
export AWS_DEFAULT_REGION="us-west-2"
export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
export TEMPOUT="$(mktemp)"
export ALIAS_VERSION="$(aws ssm get-parameter --name "/aws/service/eks/optimized-ami/${K8S_VERSION}/amazon-linux-2023/x86_64/standard/recommended/image_id" --query Parameter.Value | xargs aws ec2 describe-images --query 'Images[0].Name' --image-ids | sed -r 's/^.*(v[[:digit:]]+).*$/\1/')"


echo "${KARPENTER_NAMESPACE}" "${KARPENTER_VERSION}" "${K8S_VERSION}" "${CLUSTER_NAME}" "${AWS_DEFAULT_REGION}" "${AWS_ACCOUNT_ID}" "${TEMPOUT}" "${ALIAS_VERSION}"







3.Create cluster.yaml

apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: karpenter-demo
  region: us-west-2
iam:
  withOIDC: true
  serviceAccounts:
    - metadata:
        name: aws-load-balancer
        namespace: kube-system
      wellKnownPolicies:
        awsLoadBalancerController: true
managedNodeGroups:
  - name: demo-ng
    instanceType: t2.micro
    minSize: 1
    maxSize: 2




2.AWS CloudFormation template provisions resources required for Karpenter to function in an Amazon EKS cluster
 https://github.com/RekhuGopal/PythonHacks/blob/main/AWS_EKS_Karpenter/cloudformation.yaml
Vi Cloudformation.yaml
make cloudformation yaml



aws cloudformation deploy \
  --stack-name "Karpenter-${CLUSTER_NAME}" \
  --template-file "${TEMPOUT}" \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides "ClusterName=${CLUSTER_NAME}"








3.Add the Karpenter node role to the aws-auth configmap {role made in cloudformation.yaml file}





eksctl create iamidentitymapping \
  --username 'system:node:{{EC2PrivateDNSName}}' \
  --cluster "cluster-name" \
  --arn "arn:aws:iam::474668392434:role/KarpenterNodeRole-cluster-name" \
  --group system:bootstrappers \
  --group system:nodes

4. Create KarpenterController IAM Role

eksctl utils associate-iam-oidc-provider --cluster $CLUSTER_NAME --approve

eksctl create iamserviceaccount --cluster "$CLUSTER_NAME" --name karpenter --namespace karpenter --role-name "$CLUSTER_NAME-karpenter" --attach-policy-arn "arn:aws:iam::474668392434:policy/KarpenterControllerPolicy-karpenter-demo" --role-only --approve 


$KARPENTER_IAM_ROLE_ARN="arn:aws:iam::357171621133:role/$CLUSTER_NAME-karpenter"

eksctl get iamserviceaccount --cluster $CLUSTER_NAME --namespace karpenter




5. Create the EC2 Spot Service Linked Role

aws iam create-service-linked-role --aws-service-name spot.amazonaws.com


	
	
	
	
	
	6.Helm iNstall
	====getting endpoint
	

	export CLUSTER_ENDPOINT="https://8DFEEC4BDDCDC0000E49BE10156BF93E.gr7.ap-south-1.eks.amazonaws.com"

	
	


helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --version "$KARPENTER_VERSION_STR" \
  --namespace karpenter \
  --create-namespace \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="$KARPENTER_IAM_ROLE_ARN" \
  --set settings.clusterName="$CLUSTER_NAME" \
  --set settings.clusterEndpoint="$CLUSTER_ENDPOINT" \
  --set settings.interruptionQueue="$CLUSTER_NAME" \
  --set settings.featureGates.drift=true \
  --wait



7.Getting the image for nodepool-nodecalss
 aws ssm get-parameter --name "/aws/service/eks/optimized-ami/${K8S_VERSION}/amazon-linux-2023/x86_64/standard/recommended/image_id" --query Parameter.Value

Find image 

vi nodepool-nodeclass.yaml
We have to replace ami =-image 






K apply -f nodepool-nodeclass.yaml



8.Deploy Application

kubectl apply -f basic-deploy.yaml

10. Scale Application

kubectl scale deployment -n workshop inflate --replicas 5

kubectl -n karpenter logs -l app.kubernetes.io/name=karpenter

kubectl get node -l eks-immersion-team=my-team -o json | jq -r '.items[0].metadata.labels'




## Lab2:[Global] AWS Load Balance Controller, Ingress and test application

## If you have created EKS cluster, please ignore following step 1 and 2.
## [optional] 1.Prepare EKS cluster yaml file - eksgo04-cluster.yaml

```
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: jerry-eksworkshop
  region: ap-northeast-1
  version: "1.19"

managedNodeGroups:
  - name: ng-eksworkshop-01
    instanceType: t3.xlarge
    instanceName: ng-eksgo04
    desiredCapacity: 3
    minSize: 1
    maxSize: 8
    volumeSize: 100
    ssh:
      publicKeyName: /*your_key_pair*/
      allow: true
      enableSsm: true
    iam:
      withAddonPolicies:
        ebs: true
        fsx: true
        efs: true
```

## [optional] 2.Create EKS cluster
```
eksctl create cluster -f  eksgo04-cluster.yaml
```

## 3.Set parameters and get info
```
export CLUSTER_NAME=eksworkshop-eksctl2
export AWS_REGION=ap-northeast-1
```

Get vpc-id and subnet-id
```
[ec2-user@ip-172-31-1-111 ~]$ aws eks describe-cluster --name ${CLUSTER_NAME} | jq .cluster.resourcesVpcConfig.subnetIds,.cluster.resourcesVpcConfig.vpcId
[
  "subnet-01836074eb37c2573",
  "subnet-0bcdb907d041d08a1",
  "subnet-033a3bfd816574d36",
  "subnet-0cbe0998cc6dc4685",
  "subnet-0d2c97596faca74ad",
  "subnet-025d5ba67fdf8c030"
]
"vpc-0045fd88110566526"
```

Check Tags related to subnets
```
[ec2-user@ip-172-31-1-111 ~]$ aws ec2 describe-subnets --filters "Name=subnet-id,Values="*subnet-01836074eb37c2573*"" | jq .Subnets[0].Tags

[
  {
    "Value": "eksctl-eksworkshop-eksctl-cluster",
    "Key": "aws:cloudformation:stack-name"
  },
  {
    "Value": "0.44.0",
    "Key": "alpha.eksctl.io/eksctl-version"
  },
  {
    "Value": "SubnetPublicAPNORTHEAST1A",
    "Key": "aws:cloudformation:logical-id"
  },
  {
    "Value": "eksworkshop-eksctl",
    "Key": "eksctl.cluster.k8s.io/v1alpha1/cluster-name"
  },
  {
    "Value": "eksworkshop-eksctl",
    "Key": "alpha.eksctl.io/cluster-name"
  },
  {
    "Value": "arn:aws:cloudformation:ap-northeast-1:778108401481:stack/eksctl-eksworkshop-eksctl-cluster/c7bee820-fb1e-11eb-ac34-0ec0ff3ddc99",
    "Key": "aws:cloudformation:stack-id"
  },
  {
    "Value": "1",
    "Key": "kubernetes.io/role/elb"
  },
  {
    "Value": "eksctl-eksworkshop-eksctl-cluster/SubnetPublicAPNORTHEAST1A",
    "Key": "Name"
  }
]
```

Set every subnets in your VPC for EKS cluster to have tag

```
for NAME in $(aws eks describe-cluster --name ${CLUSTER_NAME} | jq .cluster.resourcesVpcConfig.subnetIds[])
do
  eval aws ec2 create-tags --resources ${NAME} --tags Key="kubernetes.io/cluster/${CLUSTER_NAME}",Value=shared
done
```

## 4.To allow the cluster to use AWS Identity and Access Management (IAM) for service accounts, create IAM OIDC provider

View your cluster's **OpenID Connect provider URL**(OIDC endpoint/URL is used to create an IAM OIDC identity provider for EKS cluster, determining the location of the OpenID Provider )
```
aws eks describe-cluster --name ${CLUSTER_NAME} --query "cluster.identity.oidc.issuer" --output text
```
Example output
https://oidc.eks.us-west-2.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E

List the IAM OIDC identity providers in your account.

```
aws iam list-open-id-connect-providers | grep $(aws eks describe-cluster --name ${CLUSTER_NAME} --query "cluster.identity.oidc.issuer" --output text | awk -F/ '{print $NF}')
```

Example output
```
"Arn": "arn:aws:iam::111122223333:oidc-provider/[oidc.eks.us-west-2.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E](http://oidc.eks.us-west-2.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E)"
```

If the there is no IAM OIDC identity provider, you can create an IAM OIDC identity provider ( Service Provider ) for your cluster with the following command.

```
eksctl utils associate-iam-oidc-provider --cluster=${CLUSTER_NAME} --approve --region ${AWS_REGION}
```

```
[ec2-user@ip-172-31-1-111 ekslab]$ eksctl utils associate-iam-oidc-provider --cluster=${CLUSTER_NAME} --approve --region ${AWS_REGION}
2021-08-12 07:27:02 [ℹ]  eksctl version 0.44.0
2021-08-12 07:27:02 [ℹ]  using region ap-northeast-1
2021-08-12 07:27:02 [ℹ]  will create IAM Open ID Connect provider for cluster "eksworkshop-eksctl2" in "ap-northeast-1"
2021-08-12 07:27:03 [✔]  created IAM Open ID Connect provider for cluster "eksworkshop-eksctl2" in "ap-northeast-1"
```

Check IAM OIDC identity provider again

```
[ec2-user@ip-172-31-1-111 ~]$ aws iam list-open-id-connect-providers | grep $(aws eks describe-cluster --name ${CLUSTER_NAME} --query "cluster.identity.oidc.issuer" --output text | awk -F/ '{print $NF}')
            "Arn": "arn:aws:iam::778108401481:oidc-provider/oidc.eks.ap-northeast-1.amazonaws.com/id/7D6FAF8B7A519E5494B5475F0EDAFBA1"
```

## 5.Create an IAM policy for the service account using the correct permissions 

下载IAM policy file
```
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.2.0/docs/install/iam_policy.json
```
下载cert-manager yaml file
```
curl -OL https://github.com/jetstack/cert-manager/releases/download/v1.0.2/cert-manager.yaml
```
下载AWS Load Balance Controller yaml file
```
curl -OL https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.1.3/docs/install/v2_1_3_full.yaml
```

创建名为AWSLoadBalancerControllerIAMPolicy的IAM policy
```
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```
记录返回的Plociy ARN
```
POLICY_NAME=$(aws iam list-policies --query 'Policies[?PolicyName==`AWSLoadBalancerControllerIAMPolicy`].Arn' --output text --region ${AWS_REGION})
```

## 6.To create a service account
```
eksctl create iamserviceaccount \
       --cluster=${CLUSTER_NAME} \
       --namespace=kube-system \
       --name=aws-load-balancer-controller \
       --attach-policy-arn=${POLICY_NAME} \
       --override-existing-serviceaccounts \
       --approve
```

```
[ec2-user@ip-172-31-1-111 ekslab]$ eksctl get iamserviceaccount --cluster ${CLUSTER_NAME} --name aws-load-balancer-controller --namespace kube-system
2021-08-12 07:32:48 [ℹ]  eksctl version 0.44.0
2021-08-12 07:32:48 [ℹ]  using region ap-northeast-1
NAMESPACE       NAME                            ROLE ARN
kube-system     aws-load-balancer-controller    arn:aws:iam::778108401481:role/eksctl-eksworkshop-eksctl2-addon-iamservicea-Role1-ILDU84VO7NS3
```

```
[ec2-user@ip-172-31-1-111 ekslab]$ kubectl get serviceaccounts --all-namespaces  -o wide | grep aws-load-balancer-controller

kube-system       aws-load-balancer-controller        1         94s
```

## 7.Install certManager
安装配置certManager，并检查
```
kubectl apply -f cert-manager.yaml
```
```
[ec2-user@ip-172-31-1-111 ekslab]$ kubectl get pods -n cert-manager
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-689bbbbd69-jqvph              1/1     Running   0          10s
cert-manager-cainjector-6d89f558bb-hw6q7   1/1     Running   0          10s
cert-manager-webhook-6f46d85b5f-wq7p2      1/1     Running   0          10s
```

## 8.CreateAWS Load Balance Controller
修改配置文件中的Cluster Name等
```
eval sed -i 's/your-cluster-name/${CLUSTER_NAME}/g' v2_1_3_full.yaml
sed -i '/- --ingress-class=alb/ i\            - --enable-shield=false\n            - --enable-waf=false\n            - --enable-wafv2=false' v2_1_3_full.yaml
```

检查修改好的yaml
```
[ec2-user@ip-172-31-1-111 ekslab]$ more v2_1_3_full.yaml  | grep -i Cluster-Name -A 5
            - --cluster-name=eksgo04
            - --enable-shield=false
            - --enable-waf=false
            - --enable-wafv2=false
            - --ingress-class=alb
          image: amazon/aws-alb-ingress-controller:v2.1.3
```
使用修改好的yaml文件部署AWS Load Balance Controller
```
kubectl apply -f v2_1_3_full.yaml
```
检查确认AWS Load Balance Controller是否工作
```
[ec2-user@ip-172-31-1-111 ekslab]$ kubectl logs -n kube-system $(kubectl get po -n kube-system | egrep -o aws-load-balancer-controller[a-zA-Z0-9-]+) | grep -i success
I0319 11:57:51.080612       1 leaderelection.go:252] successfully acquired lease kube-system/aws-load-balancer-controller-leader
```

## 9.Create test application
```
curl -OL https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.2.0/docs/examples/2048/2048_full.yaml
kubectl apply -f 2048_full.yaml
```

## 10.Check Ingress
```
[ec2-user@ip-172-31-1-111 ekslab]$ kubectl get ingress -A
Warning: extensions/v1beta1 Ingress is deprecated in v1.14+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
NAMESPACE   NAME           CLASS    HOSTS   ADDRESS                                                                           PORTS   AGE
game-2048   ingress-2048   <none>   *       k8s-game2048-ingress2-20ccadf1ec-1671828624.cn-northwest-1.elb.amazonaws.com.cn   80      46s
```

## 11.Troubleshooting, check logs if there is an issue
```
kubectl logs -n kube-system deployment.apps/aws-load-balancer-controller
```
If the service account doesn’t get IAM role, you can
```
[ec2-user@ip-172-31-1-111 ekslab]$ kubectl annotate serviceaccount aws-load-balancer-controller -n kube-system eks.amazonaws.com/role-arn=arn:aws-cn:iam::<your_account_id>:role/eksctl-eksgo04-addon-iamserviceaccount-kube-Role1-1JN8QIS4R9C3
serviceaccount/aws-load-balancer-controller annotated


[ec2-user@ip-172-31-1-111 ekslab]$ kubectl annotate serviceaccount aws-load-balancer-controller -n kube-system eks.amazonaws.com/role-arn=arn:aws:iam::778108401481:role/eksctl-eksworkshop-eksctl2-addon-iamservicea-Role1-ILDU84VO7NS3
```
```
[ec2-user@ip-172-31-1-111 ekslab]$ kubectl describe serviceaccounts aws-load-balancer-controller -n kube-system
Name:                aws-load-balancer-controller
Namespace:           kube-system
Labels:              app.kubernetes.io/component=controller
                     app.kubernetes.io/name=aws-load-balancer-controller
Annotations:         eks.amazonaws.com/role-arn: arn:aws-cn:iam::<your_account_id>:role/eksctl-eksgo04-addon-iamserviceaccount-kube-Role1-1JN8QIS4R9C3

Image pull secrets:  <none>
Mountable secrets:   aws-load-balancer-controller-token-gxfqg
Tokens:              aws-load-balancer-controller-token-gxfqg
Events:              <none>
```

Reference:  
https://docs.amazonaws.cn/eks/latest/userguide/aws-load-balancer-controller.html  
https://github.com/kubernetes-sigs/aws-load-balancer-controller

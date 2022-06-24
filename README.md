# Grant access to customer EKS Cluster

Sometimes you need to access customer-managed EKS cluster with public endpoint. Follow this guide to get a read-only access to customer-managed EKS cluster on customer AWS account.

## Create IAM Role (cross-account) or temporary IAM User

[AWS Cross AWS Account Access](https://docs.aws.amazon.com/IAM/latest/UserGuide/tutorial_cross-account-with-roles.html)

In order to access EKS cluster you need AWS IAM credentials. The most secure way is to define a cross AWS account role and assume this role.

### Cross-account IAM Role

[![Launch Stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home#/stacks/new?stackName=eks-ro-acccess&templateURL=https://min.gitcdn.link/cdn/alexei-led/eks-ro-access/master/template.yaml)

CloudFormation [template](./template.yaml) for read-only access to an EKS cluster.

### Using temporary IAM User

Another, less secure option, is to create a temporary IAM User in customer AWS account.

Then you need to attach IAM Policy to cross-account IAM Role or in-account IAM User.

The required IAM Policy (replace `<>` values):

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": ["eks:DescribeCluster", "eks:ListClusters"],
            "Resource": "arn:aws:eks:<AWS_REGION>:<AWS_ACCOUNT>:cluster/<EKS_NAME>"
        }
    ]
}
```

## Create ClusterRoleBinding to `view` ClusterRole

[Kubernetes RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

EKS has a built-in `view` ClusterRole with `get,list,watch` access to all APIs and all resources.

Please, create a `support:viewer` ClusterRoleBinding to the `view` ClusterRole.

```sh
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  name: support:viewer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: support:viewer
EOF
```

## Add IAM role/user to aws-auth ConfigMap

[AWS guide](https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html)

Edit `aws-auth` ConfigMap manually (`kubectl edit -n kube-system configmap/aws-auth` command) or with `eksctl create iamidentitymapping`, adding IAM User or IAM Role to `mapUsers` or `mapRoles` configuration.

For example:

```sh
eksctl create iamidentitymapping --username viewer --group support:viewer --arn <USER_ARN|ROLE_ARN> --cluster <CLUSTER_NAME> --region <AWS_REGION>
```

## Generate/update kubeconfig

[AWS guide](https://docs.aws.amazon.com/eks/latest/userguide/create-kubeconfig.html)

Generate/update `kubeconfig` for the EKS clusterm assuming IAM Role (from above) or using AWS credentials for temporary IAM User.

```sh
aws eks --region <AWS_REGION> update-kubeconfig --name <CLUSTER_NAME>
```

## Additional References

- [How do I manage permissions across namespaces for my IAM users in an Amazon EKS cluster?](https://aws.amazon.com/premiumsupport/knowledge-center/eks-iam-permissions-namespaces/)
- [Read-Only Access to Kubernetes Cluster](https://medium.com/@rschoening/read-only-access-to-kubernetes-cluster-fcf84670b698)
- [eksctl: Manage IAM users and roles](https://eksctl.io/usage/iam-identity-mappings/)
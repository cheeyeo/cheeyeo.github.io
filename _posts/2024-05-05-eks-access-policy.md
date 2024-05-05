---
layout: post
show_meta: true
title: Managing EKS cluster access
header: Managing EKS cluster access
date: 2024-05-05 00:00:00
summary: Using EKS access policy to manage EKS cluster access
categories: aws eks kubernetes
author: Chee Yeo
---

[Kubernetes user-facing roles]: https://kubernetes.io/docs/reference/access-authn-authz/rbac/#user-facing-roles
[Managing EKS access entries]: https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html

EKS is a managed service from AWS for creating kubernetes clusters. AWS manages the control plane components while the cluster access is managed by the end user. This is normally a 2 step process:

* Create the appropriate IAM role with the right permissions
* Create the ClusterRole / Role and its role bindings
* Update the aws-auth configmap to associate the IAM role with the role group created in step 2

This is often a manual process even when using IAC as we would have to create the cluster first and then run something like kustomize to update the aws-auth configmap. It also introduces tight coupling between the IAM role and the aws-auth configmap - any changes made to the IAM role would require rebuilding the aws-auth configmap. We also need to add additional permissions to the IAM role to allow access to say the EKS console in the dashboard.

With the latest EKS versions ( 1.29+ ), we can use access policies which are based on [Kubernetes user-facing roles] to create an access entry, which links the IAM role to the specified cluster role.


To view the list of access policies:

{% highlight shell %}
    aws eks list-access-policies
{% endhighlight %}

For example, given an IAM role of `KubernetesAdmin`, we can run the following CLI command to associate it with the `AmazonEKSClusterAdminPolicy`, which grants cluster-wide admin access:

{% highlight shell %}
aws eks create-access-entry --cluster-name <CLUSTER_NAME> \
  --principal-arn <IAM_PRINCIPAL_ARN>


aws eks associate-access-policy --cluster-name <CLUSTER_NAME> \
  --principal-arn <IAM_PRINCIPAL_ARN> \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy
  --access-scope type=cluster
{% endhighlight %}

In terraform using the `eks` module:
{% highlight terraform %}
module "eks" {
    source  = "terraform-aws-modules/eks/aws"
    version = "20.8.5"

    ...

    access_entries = {
        eks-admin = {
            kubernetes_groups = []
            principal_arn     = aws_iam_role.kubernetes_admin.arn

            policy_associations = {
                single = {
                    policy_arn = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy"
                    access_scope = {
                        namespaces = []
                        type       = "cluster"
                    }
                }
            }
        }
    }
}
{% endhighlight %}

To view the access entries, we can click on the cluster in EKS console and navigate to `Access` or via the cli:
{% highlight shell %}
aws eks list-access-entries --cluster-name mycluster
{% endhighlight %}

From the deployment above, the following entries were created:
{% highlight shell %}
{
    "accessEntries": [
        "arn:aws:iam::XXXX:role/KubernetesAdmin",
        "arn:aws:iam::XXXX:role/node-group-1-eks-node-group-XXXX",
        "arn:aws:iam::XXXX:role/node-group-2-eks-node-group-XXXX",
    ]
}
{% endhighlight %}

From above, we can see that the `KubernetesAdmin` role has been added successfully to the cluster access entries. To test the assignment, we can generate the kubeconfig for the role and try to access the cluster:

{% highlight shell %}
aws eks update-kubeconfig --region eu-west-1 --name mycluster --role-arn arn:aws:iam::XXXX:role/KubernetesAdmin

kubectl cluster-info
{% endhighlight %}

We can try to perform some cluster admin tasks by viewing and creating resources:
{% highlight shell %}
kubectl auth can-i delete pods -A

kubectl auth can-i get configmap/aws-auth -n kube-system

kubectl get configmaps -A

kubectl get secrets -A

kubectl -n kube-system create secret generic testsecret

kubectl -n kube-system delete secret testsecret
{% endhighlight %}

Personally, I find that using the access entries and policies to be easier to understand in terms of access management as we don't have to manually update the aws-auth configmap.

More details can be found on [Managing EKS access entries].
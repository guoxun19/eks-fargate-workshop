

# 创建 EKS 集群



在创建 EKS 集群之前，我们还需要安装 eksctl 和 kubectl 工具。



## 安装 kubectl

```bash
sudo curl --silent --location -o /usr/local/bin/kubectl \
  https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl

sudo chmod +x /usr/local/bin/kubectl
```

参考文档：https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html

验证 kubectl 安装结果：

```bash
kubectl version --client
```



## 安装 eksctl

[eksctl](https://eksctl.io/) 是 Amazon EKS 的官方管理工具, Go 语言实现, 底层通过 CloudFormation 对 AWS 资源进行管理。

安装命令:

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

sudo mv -v /tmp/eksctl /usr/local/bin
```

参考文档：https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html

验证 eksctl 安装结果:

```bash
eksctl version
```



## 通过 eksctl 创建 EKS 集群

下面的命令会创建一个名字为 **eks-fargate** 的 EKS 集群, 这个 EKS 集群会创建一个默认的 [Fargate Profile](https://docs.aws.amazon.com/eks/latest/userguide/fargate-profile.html)，同时也会创建 EC2 工作节点。

在创建集群的同时，会自动创建新的 VPC。后续创建的 Fargate 任务会运行在这个新创建的 VPC 中。也可以通过配置指定使用已有的 VPC。

```bash
eksctl create cluster \
--name eks-fargate \
--version 1.18 \
--nodegroup-name mg-nodegroup-1 \
--nodes 1 \
--nodes-min 1 \
--nodes-max 2 \
--with-oidc \
--managed \
--fargate
```



也可以创建只有运行 Fargate 的集群，不创建 EC2 工作节点组。

```bash
eksctl create cluster --name eks-fargate --without-nodegroup --fargate --with-oidc
```





整个 EKS 集群创建过程大约 15 分钟, 输出信息如下图所示:

```bash
2021-03-30 15:34:47 [ℹ]  eksctl version 0.42.0
2021-03-30 15:34:47 [ℹ]  using region ap-southeast-1
2021-03-30 15:34:47 [ℹ]  setting availability zones to [ap-southeast-1b ap-southeast-1a ap-southeast-1c]
2021-03-30 15:34:47 [ℹ]  subnets for ap-southeast-1b - public:192.168.0.0/19 private:192.168.96.0/19
2021-03-30 15:34:47 [ℹ]  subnets for ap-southeast-1a - public:192.168.32.0/19 private:192.168.128.0/19
2021-03-30 15:34:47 [ℹ]  subnets for ap-southeast-1c - public:192.168.64.0/19 private:192.168.160.0/19
2021-03-30 15:34:47 [ℹ]  using Kubernetes version 1.18
2021-03-30 15:34:47 [ℹ]  creating EKS cluster "eks-fargate" in "ap-southeast-1" region with Fargate profile and managed nodes
2021-03-30 15:34:47 [ℹ]  will create 2 separate CloudFormation stacks for cluster itself and the initial managed nodegroup
2021-03-30 15:34:47 [ℹ]  if you encounter any issues, check CloudFormation console or try 'eksctl utils describe-stacks --region=ap-southeast-1 --cluster=eks-fargate'
2021-03-30 15:34:47 [ℹ]  CloudWatch logging will not be enabled for cluster "eks-fargate" in "ap-southeast-1"
2021-03-30 15:34:47 [ℹ]  you can enable it with 'eksctl utils update-cluster-logging --enable-types={SPECIFY-YOUR-LOG-TYPES-HERE (e.g. all)} --region=ap-southeast-1 --cluster=eks-fargate'
2021-03-30 15:34:47 [ℹ]  Kubernetes API endpoint access will use default of {publicAccess=true, privateAccess=false} for cluster "eks-fargate" in "ap-southeast-1"
2021-03-30 15:34:47 [ℹ]  2 sequential tasks: { create cluster control plane "eks-fargate", 3 sequential sub-tasks: { 5 sequential sub-tasks: { wait for control plane to become ready, create fargate profiles, associate IAM OIDC provider, 2 sequential sub-tasks: { create IAM role for serviceaccount "kube-system/aws-node", create serviceaccount "kube-system/aws-node" }, restart daemonset "kube-system/aws-node" }, create addons, create managed nodegroup "mg-nodegroup-1" } }
2021-03-30 15:34:47 [ℹ]  building cluster stack "eksctl-eks-fargate-cluster"
2021-03-30 15:34:48 [ℹ]  deploying stack "eksctl-eks-fargate-cluster"
2021-03-30 15:35:18 [ℹ]  waiting for CloudFormation stack "eksctl-eks-fargate-cluster"
...
2021-03-30 15:46:49 [ℹ]  waiting for CloudFormation stack "eksctl-eks-fargate-cluster"
2021-03-30 15:46:50 [ℹ]  creating Fargate profile "fp-default" on EKS cluster "eks-fargate"
2021-03-30 15:49:01 [ℹ]  created Fargate profile "fp-default" on EKS cluster "eks-fargate"
2021-03-30 15:49:02 [ℹ]  "coredns" is now schedulable onto Fargate
2021-03-30 15:51:09 [ℹ]  "coredns" is now scheduled onto Fargate
2021-03-30 15:51:09 [ℹ]  "coredns" pods are now scheduled onto Fargate
2021-03-30 15:51:11 [ℹ]  building iamserviceaccount stack "eksctl-eks-fargate-addon-iamserviceaccount-kube-system-aws-node"
2021-03-30 15:51:11 [ℹ]  deploying stack "eksctl-eks-fargate-addon-iamserviceaccount-kube-system-aws-node"
...
2021-03-30 15:51:44 [ℹ]  waiting for CloudFormation stack "eksctl-eks-fargate-addon-iamserviceaccount-kube-system-aws-node"
2021-03-30 15:51:44 [ℹ]  serviceaccount "kube-system/aws-node" already exists
2021-03-30 15:51:44 [ℹ]  updated serviceaccount "kube-system/aws-node"
2021-03-30 15:51:44 [ℹ]  daemonset "kube-system/aws-node" restarted
2021-03-30 15:51:45 [ℹ]  building managed nodegroup stack "eksctl-eks-fargate-nodegroup-mg-nodegroup-1"
2021-03-30 15:51:45 [ℹ]  deploying stack "eksctl-eks-fargate-nodegroup-mg-nodegroup-1"
2021-03-30 15:51:45 [ℹ]  waiting for CloudFormation stack "eksctl-eks-fargate-nodegroup-mg-nodegroup-1"
...
2021-03-30 15:55:10 [ℹ]  waiting for CloudFormation stack "eksctl-eks-fargate-nodegroup-mg-nodegroup-1"
2021-03-30 15:55:10 [ℹ]  waiting for the control plane availability...
2021-03-30 15:55:10 [✔]  saved kubeconfig as "/home/ec2-user/.kube/config"
2021-03-30 15:55:10 [ℹ]  no tasks
2021-03-30 15:55:10 [✔]  all EKS cluster resources for "eks-fargate" have been created
2021-03-30 15:55:11 [ℹ]  nodegroup "mg-nodegroup-1" has 1 node(s)
2021-03-30 15:55:11 [ℹ]  node "ip-192-168-50-173.ap-southeast-1.compute.internal" is ready
2021-03-30 15:55:11 [ℹ]  waiting for at least 1 node(s) to become ready in "mg-nodegroup-1"
2021-03-30 15:55:11 [ℹ]  nodegroup "mg-nodegroup-1" has 1 node(s)
2021-03-30 15:55:11 [ℹ]  node "ip-192-168-50-173.ap-southeast-1.compute.internal" is ready
2021-03-30 15:55:12 [ℹ]  kubectl command should work with "/home/ec2-user/.kube/config", try 'kubectl get nodes'
2021-03-30 15:55:12 [✔]  EKS cluster "eks-fargate" in "ap-southeast-1" region is ready
```



验证集群工作状态:

```bash
$ kubectl get nodes
NAME                                                         STATUS   ROLES    AGE     VERSION
fargate-ip-192-168-115-121.ap-southeast-1.compute.internal   Ready    <none>   6m53s   v1.18.9-eks-866667
fargate-ip-192-168-155-231.ap-southeast-1.compute.internal   Ready    <none>   6m56s   v1.18.9-eks-866667
ip-192-168-50-173.ap-southeast-1.compute.internal            Ready    <none>   2m54s   v1.18.9-eks-d1db3c
```

可以看到现在有两个 Fargate 节点在运行，分别承载的是两个 coreDNS Pod。

```bash
$ kubectl get po --all-namespaces -o wide
NAMESPACE     NAME                       READY   STATUS    RESTARTS   AGE     IP                NODE                                                         NOMINATED NODE   READINESS GATES
kube-system   aws-node-mrnwz             1/1     Running   0          3m44s   192.168.50.173    ip-192-168-50-173.ap-southeast-1.compute.internal            <none>           <none>
kube-system   coredns-85c7d498b7-bbhmt   1/1     Running   0          8m34s   192.168.155.231   fargate-ip-192-168-155-231.ap-southeast-1.compute.internal   <none>           <none>
kube-system   coredns-85c7d498b7-qh9wd   1/1     Running   0          8m34s   192.168.115.121   fargate-ip-192-168-115-121.ap-southeast-1.compute.internal   <none>           <none>
kube-system   kube-proxy-lwt8p           1/1     Running   0          3m44s   192.168.50.173    ip-192-168-50-173.ap-southeast-1.compute.internal            <none>           <none>
```



查看 eksctl 为我们创建的默认 Fargate Profile，它指定了 default 和 kube-system 两个 namespace 中的 pod 都会运行在 Fargate 上。

```bash
eksctl get fargateprofile --cluster eks-fargate -o yaml
```

输出的 Fargate Profile 信息示例如下：

```yaml
- name: fp-default
  podExecutionRoleARN: arn:aws:iam::958730362563:role/eksctl-eks-fargate-cluster-FargatePodExecutionRole-3651GPAEK29W
  selectors:
  - namespace: default
  - namespace: kube-system
  status: ACTIVE
  subnets:
  - subnet-06f96b0140cea28fb
  - subnet-03d244d868c8f7b32
  - subnet-09b1c4f197e625642
```



至此，我们成功的完成了 Fargate 集群的创建。




# 步骤3 - 部署 AWS LoadBalancer Controller



[AWS Load Balancer Controller](https://github.com/kubernetes-sigs/aws-load-balancer-controller) 是一个控制器，可帮助管理 Kubernetes 集群的 Elastic Load Balancer。

- 它通过部署和配置 Application Load Balancers - ALB 来提供 Kubernetes Ingress 资源。
- 它通过部署和配置 Network Load Balancers - NLB 来提供 Kubernetes Service 资源。



AWS ALB 和 NLB 可以和部署在 Fargate 上的 Service/Ingress 进行集成，通过 IP 模式将流量从负载均衡器直接转发到 Pod IP 上。其中，

- 对于 Kubernetes Service，可以使用 AWS 网络负载均衡器 (NLB，IP 模式) 对跨 Pod 的 4 层网络流量进行负载均衡。
- 对于 Kubernetes Ingress，可以使用 AWS 应用负载均衡器 (ALB，IP 模式) 对跨 Pod 的 7 层网络流量进行负载均衡。



参考资料：https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html

### 创建 IAM 策略

下载 IAM 策略文件，用于为 AWS Load Balancer Controller 配置 [IRSA](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) 权限 

```bash
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.1.3/docs/install/iam_policy.json
```



使用下载到策略文件创建 IAM 策略

```bash
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```



### 创建 IAM role 和 service account

为 AWS Load Balancer Controller 创建一个 IAM role 和 Service Account 并将两者进行关联，以便 AWS Load Balancer Controller 所在 Pod 拥有相应的 IAM 权限（如创建 ELB、TargetGroup 等）。

执行如下命令，注意将 cluster 名称和 policy-arn 替换成你自己的值。

```bash
eksctl create iamserviceaccount \
  --cluster=<CLUSTER_NAME> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve
```



创建成功后 eksctl 输出示例如下：

```bash
2021-03-30 16:00:32 [ℹ]  eksctl version 0.42.0
2021-03-30 16:00:32 [ℹ]  using region ap-southeast-1
2021-03-30 16:00:33 [ℹ]  1 existing iamserviceaccount(s) (kube-system/aws-node) will be excluded
2021-03-30 16:00:33 [ℹ]  1 iamserviceaccount (kube-system/aws-load-balancer-controller) was included (based on the include/exclude rules)
2021-03-30 16:00:33 [!]  metadata of serviceaccounts that exist in Kubernetes will be updated, as --override-existing-serviceaccounts was set
2021-03-30 16:00:33 [ℹ]  1 task: { 2 sequential sub-tasks: { create IAM role for serviceaccount "kube-system/aws-load-balancer-controller", create serviceaccount "kube-system/aws-load-balancer-controller" } }
2021-03-30 16:00:33 [ℹ]  building iamserviceaccount stack "eksctl-eks-fargate-addon-iamserviceaccount-kube-system-aws-load-balancer-controller"
2021-03-30 16:00:33 [ℹ]  deploying stack "eksctl-eks-fargate-addon-iamserviceaccount-kube-system-aws-load-balancer-controller"
2021-03-30 16:00:33 [ℹ]  waiting for CloudFormation stack "eksctl-eks-fargate-addon-iamserviceaccount-kube-system-aws-load-balancer-controller"
2021-03-30 16:00:50 [ℹ]  waiting for CloudFormation stack "eksctl-eks-fargate-addon-iamserviceaccount-kube-system-aws-load-balancer-controller"
2021-03-30 16:01:06 [ℹ]  waiting for CloudFormation stack "eksctl-eks-fargate-addon-iamserviceaccount-kube-system-aws-load-balancer-controller"
2021-03-30 16:01:07 [ℹ]  created serviceaccount "kube-system/aws-load-balancer-controller"
```



创建完成后，可以开始部署 AWS Load Balancer Controller。



### 部署 AWS Load Balancer Controller

可以通过 Helm 或者手工的方式部署，在本次实验中我们使用 Helm 的方式。

#### 安装 Helm

```bash
curl -sSL https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```

验证 Helm 安装

```bash
helm version --short
```



#### 安装 Controller

首先安装 TargetGroupBinding 这个 CRD

```bash
kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"
```

添加 eks-chart

```bash
helm repo add eks https://aws.github.io/eks-charts
```

使用 Helm 安装 Controller，将 cluster-name 替换成你的集群名称，region-name 替换成你的 EKS 集群所在 region，vpc-id 替换成你的 EKS 集群所在 VPC ID

```bash
helm upgrade -i aws-load-balancer-controller eks/aws-load-balancer-controller \
  --set clusterName=eks-fargate \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<region-name> \
  --set vpcId=<vpc-id> \
  -n kube-system
```



查看部署状态和日志

```bash
$ kubectl get deployment -n kube-system aws-load-balancer-controller
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
aws-load-balancer-controller   1/1     1            1           3m21s

$ kubectl get po  -n kube-system | grep load-balancer
aws-load-balancer-controller-55887567d7-sk5xt   1/1     Running   0          3m59s
```

查看 Controller 日志，注意将 pod name 替换成你自己的 Controller pod 名称

```bash
$ kubectl logs -n kube-system aws-load-balancer-controller-55887567d7-sk5xt
{"level":"info","ts":1617121325.3599958,"msg":"version","GitVersion":"v2.1.3","GitCommit":"c9d30f23960f12cd1e985e9b2cd3f077b9a8c93f","BuildDate":"2021-02-18T19:32:05+0000"}
{"level":"info","ts":1617121325.465933,"logger":"controller-runtime.metrics","msg":"metrics server is starting to listen","addr":":8080"}
{"level":"info","ts":1617121325.554983,"logger":"setup","msg":"adding health check for controller"}
{"level":"info","ts":1617121325.5550823,"logger":"controller-runtime.webhook","msg":"registering webhook","path":"/mutate-v1-pod"}
{"level":"info","ts":1617121325.555134,"logger":"controller-runtime.webhook","msg":"registering webhook","path":"/mutate-elbv2-k8s-aws-v1beta1-targetgroupbinding"}
{"level":"info","ts":1617121325.5551763,"logger":"controller-runtime.webhook","msg":"registering webhook","path":"/validate-elbv2-k8s-aws-v1beta1-targetgroupbinding"}
{"level":"info","ts":1617121325.5563838,"logger":"setup","msg":"starting podInfo repo"}
{"level":"info","ts":1617121327.5565908,"logger":"controller-runtime.manager","msg":"starting metrics server","path":"/metrics"}
I0330 16:22:07.556529       1 leaderelection.go:242] attempting to acquire leader lease  kube-system/aws-load-balancer-controller-leader...
I0330 16:22:07.578003       1 leaderelection.go:252] successfully acquired lease kube-system/aws-load-balancer-controller-leader
{"level":"info","ts":1617121327.6570337,"logger":"controller-runtime.webhook.webhooks","msg":"starting webhook server"}
{"level":"info","ts":1617121327.657261,"logger":"controller","msg":"Starting EventSource","reconcilerGroup":"elbv2.k8s.aws","reconcilerKind":"TargetGroupBinding","controller":"targetGroupBinding","source":"kind source: /, Kind="}
{"level":"info","ts":1617121327.657538,"logger":"controller","msg":"Starting EventSource","reconcilerGroup":"elbv2.k8s.aws","reconcilerKind":"TargetGroupBinding","controller":"targetGroupBinding","source":"kind source: /, Kind="}
{"level":"info","ts":1617121327.657676,"logger":"controller","msg":"Starting EventSource","reconcilerGroup":"elbv2.k8s.aws","reconcilerKind":"TargetGroupBinding","controller":"targetGroupBinding","source":"kind source: /, Kind="}
{"level":"info","ts":1617121327.6579278,"logger":"controller-runtime.certwatcher","msg":"Updated current TLS certificate"}
{"level":"info","ts":1617121327.6585114,"logger":"controller-runtime.webhook","msg":"serving webhook server","host":"","port":9443}
{"level":"info","ts":1617121327.6578932,"logger":"controller","msg":"Starting EventSource","controller":"ingress","source":"channel source: 0xc0006f8050"}
{"level":"info","ts":1617121327.6586843,"logger":"controller","msg":"Starting EventSource","controller":"ingress","source":"channel source: 0xc0006f80a0"}
{"level":"info","ts":1617121327.6587071,"logger":"controller","msg":"Starting EventSource","controller":"ingress","source":"kind source: /, Kind="}
{"level":"info","ts":1617121327.6587658,"logger":"controller","msg":"Starting EventSource","controller":"ingress","source":"kind source: /, Kind="}
{"level":"info","ts":1617121327.6587937,"logger":"controller","msg":"Starting EventSource","controller":"ingress","source":"kind source: /, Kind="}
{"level":"info","ts":1617121327.6591783,"logger":"controller-runtime.certwatcher","msg":"Starting certificate watcher"}
{"level":"info","ts":1617121327.6579947,"logger":"controller","msg":"Starting EventSource","controller":"service","source":"kind source: /, Kind="}
{"level":"info","ts":1617121327.6592298,"logger":"controller","msg":"Starting Controller","controller":"service"}
{"level":"info","ts":1617121327.7586687,"logger":"controller","msg":"Starting EventSource","reconcilerGroup":"elbv2.k8s.aws","reconcilerKind":"TargetGroupBinding","controller":"targetGroupBinding","source":"kind source: /, Kind="}
{"level":"info","ts":1617121327.7590826,"logger":"controller","msg":"Starting Controller","controller":"ingress"}
{"level":"info","ts":1617121327.7598803,"logger":"controller","msg":"Starting workers","controller":"service","worker count":3}
{"level":"info","ts":1617121327.859231,"logger":"controller","msg":"Starting Controller","reconcilerGroup":"elbv2.k8s.aws","reconcilerKind":"TargetGroupBinding","controller":"targetGroupBinding"}
{"level":"info","ts":1617121327.8595572,"logger":"controller","msg":"Starting workers","reconcilerGroup":"elbv2.k8s.aws","reconcilerKind":"TargetGroupBinding","controller":"targetGroupBinding","worker count":3}
{"level":"info","ts":1617121327.8595293,"logger":"controller","msg":"Starting workers","controller":"ingress","worker count":3}
```










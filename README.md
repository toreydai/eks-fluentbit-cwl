### 1 配置工作线程节点的权限

获取工作线程节点Role ARN

```
STACK_NAME=$(eksctl get nodegroup --cluster eksworkshop -o json | jq -r '.[].StackName')
ROLE_NAME=$(aws cloudformation describe-stack-resources --stack-name $STACK_NAME | jq -r '.StackResources[] | select(.ResourceType=="AWS::IAM::Role") | .PhysicalResourceId')
echo "export ROLE_NAME=${ROLE_NAME}" | tee -a ~/.bash_profile
```

创建权限Policy文件，主要是CloudWatch Logs权限

```
cat <<EoF > ./eks-fluent-bit-daemonset-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "logs:PutLogEvents",
            "Resource": "arn:aws:logs:*:*:log-group:*:*:*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogStream",
                "logs:DescribeLogStreams",
                "logs:PutLogEvents"
            ],
            "Resource": "arn:aws:logs:*:*:log-group:*"
        },
        {
            "Effect": "Allow",
             "Action": [
                "logs:CreateLogGroup",
                "logs:DescribeLogGroups"
            ],
            "Resource": "*"
        }
    ]
}
EoF
```
为工作线程节点Role增加CloudWatch Logs权限

```
aws iam put-role-policy --role-name $ROLE_NAME \
--policy-name Logs-Policy-For-Worker \
--policy-document file://./eks-fluent-bit-daemonset-policy.json
```

查看Policy是否已附加到工作线程节点Role

```
aws iam get-role-policy --role-name $ROLE_NAME \
--policy-name Logs-Policy-For-Worker
```
### 2 部署Fluent Bit
执行如下命令，创建名为“amazon-cloudwatch”的命名空间

```
kubectl apply -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/ \
k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/cloudwatch-namespace.yaml
```
执行如下命令，创建名为“fluent-bit-cluster-info”的configmap
<br>需要将cluster-name和cluster-region替换成当前的值

```
ClusterName='cluster-name'
RegionName='cluster-region'
FluentBitHttpPort='2020'
FluentBitReadFromHead='Off'
[[ ${FluentBitReadFromHead} = 'On' ]] && FluentBitReadFromTail='Off'|| FluentBitReadFromTail='On'
[[ -z ${FluentBitHttpPort} ]] && FluentBitHttpServer='Off' || FluentBitHttpServer='On'

kubectl create configmap fluent-bit-cluster-info \
--from-literal=cluster.name=${ClusterName} \
--from-literal=http.server=${FluentBitHttpServer} \
--from-literal=http.port=${FluentBitHttpPort} \
--from-literal=read.head=${FluentBitReadFromHead} \
--from-literal=read.tail=${FluentBitReadFromTail} \
--from-literal=logs.region=${RegionName} -n amazon-cloudwatch
```
执行如下命令，部署Fluent Bit DaemonSet
```
kubectl apply -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest \
/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/fluent-bit/fluent-bit.yaml
```

### 3 在Amazon CloudWatch中查看日志
在Amazon CloudWatch控制台中，依次打开日志->日志组，查看是否存在如下日志组：
```
/aws/containerinsights/Cluster_Name/application
/aws/containerinsights/Cluster_Name/host
/aws/containerinsights/Cluster_Name/dataplane
```
如果存在，说明配置成功
 
### 4 清理环境
执行如下命令清理环境

```
kubectl delete -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest \
/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/fluent-bit/fluent-bit.yaml

kubectl delete cm fluent-bit-cluster-info -n amazon-cloudwatch

kubectl delete ns amazon-cloudwatch
```

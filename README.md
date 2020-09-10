# 实验环境准备
Openshift 4.4 (kubernets 1.17) 

https://console-openshift-console.apps.ocp4-beijing-df27-ipi.azure.opentlc.com


客户端 for linux: 
| Command | 安装 |备注 
| --- | --- |---|
| oc/kubectl| https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.4.3/openshift-client-linux-4.4.3.tar.gz| source<(oc completion bash)
|kn| https://github.com/knative/client/releases/tag/v0.14.0|source<(kn completion bash)
|http| dnf install httpie| http :8080 greeting="good morning"
|hey|https://storage.googleapis.com/jblabs/dist/hey_linux_v0.1.2 |hey -c 50 -z 10s $(url)
|stern|https://github.com/wercker/stern/releases/download/1.6.0/stern_linux_amd64|stern podname -c  user-container
|tkn| https://mirror.openshift.com/pub/openshift-v4/clients/pipeline/0.9.0/tkn-linux-amd64-0.9.0.tar.gz|source <(tkn completion bash)

# 实验 1 knative serving 自动伸缩实验
oc new-project serverlesslab1

检查 serving api
kubectl api-resources --api-group serving.knative.dev

## 根据Request 进行伸缩
kubectl apply -f lab1/service.yaml

测试服务已经可以访问，并且自动伸缩为0

hey -c 50 -z 10s '${SVC_URL}/?sleep=3&upto=10000&memload=100'

(注意这里一定要引号，否则bash中无法正确解析&)

检查ksvc/spec/template/metadata/annotations 标记值

增加伸缩时最小的pod数量

```yaml
spec:
  template:
    metadata:
      annotations:
        # the minimum number of pods to scale down to
        autoscaling.knative.dev/minScale: "2"
        # the maximum number of pods to scale up to
        autoscaling.knative.dev/maxScale: "5"        
        # Target 10 in-flight-requests per pod.
        autoscaling.knative.dev/target: "10"
```
查看 autoscaler的配置

kubectl -n knativetutorial describe cm config-autoscaler

使用kn命令行进行操作

kn service create|delete|update 

kubectl edit ksvc svc_name 查看

流量切换
```bash
kn service create greeter --image quay.io/rhdevelopers/knative-tutorial-greeter:quarkus --revision-name greeter-v1
kn service update greeter --image quay.io/rhdevelopers/knative-tutorial-greeter:quarkus --revision-name greeter-v2 --env MESSAGE_PREFIX=GreeterV2
kn service update greeting --traffic greeting-greeter-v1=30 --traffic greeting-greeter-v2=70 --traffic @latest=0
kn revision list

```
(注意revision名字要匹配上)


根据CPU metrics进行伸缩
```bash
kn service update prime-generator \
   --annotation autoscaling.knative.dev/minScale- \
   --annotation autoscaling.knative.dev/maxScale- \
   --annotation autoscaling.knative.dev/target=70 \
   --annotation autoscaling.knative.dev/metric=cpu \
   --annotation autoscaling.knative.dev/class=hpa.autoscaling.knative.dev
```

清理本实验

kubectl delete ksvc greeter

kubectl delete ksvc prime-generator

# 实验 2 knative pipeline
kubectl create ns  servelesslab2

确认pipeline已经装好

kubectl get sa pipeline

注意事项
请检查 你当前的namespace是否是 serverlesslab2, 如果使用不同的名称，需要改lab2里面的脚本
因为image push需要用到namespace名称
## 定义一个Pipeline 
从源代码编译 到 serverless部署

 * 第一步 Pipeline 三要素
 ```
    kubectl apply -f lab2/01_apply_manifest_task.yaml
    kubectl apply -f lab2/02_update_deployment_task.yaml
    kubectl apply -f lab2/03_resources.yaml
    kubectl apply -f lab2/04_pipeline.yaml

```
 * 第二步 手动触发Pipeline
 
 tkn pipeline start build-and-deploy
填参数

## Git push 自动触发 pipeline 运行
    * 第一步配置Trigger Template + 模板
```
    kubectl apply -f lab2/11_trigger_template.yaml 
    kubectl apply -f lab2/12_trigger_binding.yaml
```
    * 第二步 配置Event listener，监听来自git repo的事件

```bash
    kubectl apply -f lab2/13_event_listener.yaml 
```
    * 第三步 给git repo 配置webhook

```bash
    oc expose svc el-vote-app
    oc get route
    git commit -m "empty-commit" --allow-empty
```
    把 http://vote-ui-serverlesslab2.apps.ocp4-beijing-df27-ipi.azure.opentlc.com 配置到对应的webhook上面 以便接受git push的事件


# 实验 3 knative eventing I

## 模式1 source 直接触发 Service 
## 模式2 使用Channel保存和群发(订阅)消息
## 模式3 使用Broker 基于元数据分发消息

# 实验 4 knative evening II(使用kafka存储消息)
## 使用kafka进行事件保存与订阅


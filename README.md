# 实验环境准备


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
kubectl create ns serverlesslab1

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
```
kubectl delete ksvc greeter
kubectl delete ksvc prime-generator
```
# 实验 2 knative pipeline
kubectl create ns  servelesslab2

确认pipeline已经装好

kubectl get sa pipeline

注意事项
请检查 你当前的namespace是否是 serverlesslab2, 如果使用不同的名称，需要改lab2里面的脚本
因为image push需要用到namespace名称
## 定义一个Pipeline 
从源代码编译 到 serverless部署
### RTP
定义一个原生Pipeline就是定义Resource, Task, Pipeline
### Resource
    * Type -类型， 包括: git; image
    * Params - 参数
### Task RPS
    * Resources: 绑定资源参数
    * Params: 额外参数 
    * Steps: 步骤 类似Ansible Command
### Pipeline
    * Resources
    * TaskRef
 * 第一步 Pipeline 三要素 
 ```
    kubectl apply -f lab2/01_apply_manifest_task.yaml
    kubectl apply -f lab2/02_update_deployment_task.yaml
    kubectl apply -f lab2/03_resources.yaml
    kubectl apply -f lab2/04_pipeline.yaml

```
 * 第二步 手动触发Pipeline
 ```
 tkn pipeline start build-and-deploy
 >填参数
```

## Git push 自动触发 pipeline 运行
### TBL
* TriggerTemplate 就是把 PipelineResource 参数化
* TriigerBinding 就是把 把git push event 和TriggerTemplate厘米的参数 进行绑定
* EventListener 就是接受git push event然后实例化pipelineresource 并且启动pipelinerun
### 实验步骤
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
在这个实验中，学习knative eventing 接收，保存，转发，订阅,路由
```
kubectl create ns serverlesslab3
```
## 确认knative eventing api
```bash
kubectl api-resources --api-group='sources.knative.dev'
```
默认安装的是
```NAME               SHORTNAMES   APIGROUP              NAMESPACED   KIND
apiserversources                sources.knative.dev   true         ApiServerSource
pingsources                     sources.knative.dev   true         PingSource
sinkbindings                    sources.knative.dev   true         SinkBinding
```

**注意事项
请检查 你当前的namespace是否是 serverlesslab3, 如果使用不同的名称，需要改lab3里面的脚本**

## 模式1 source 直接触发 Service 
![1-event-direct.png](image/1-event-direct.png "Event direct")
* 第一步定义 source 和sink
```
source 和sink是配对出现的
Sink 可以是一个k服务,也可以是一个k Channel 这里我们定义一个k服务为sink
kubectl apply -f lab3/01-eventinghello-ping-source.yaml
```
* 第二步定义k服务
```
kubectl apply -f lab3/02-eventinghello-service.yaml
```
* 第三步 观察eventing 触发 kservice
```
#观察Cloud event的原数据信息
stern eventinghello -c user-container
```
* 清理
```
kn source list
kn source ping delete eventinghello-ping-source
```
## 模式2 使用Channel保存和群发(订阅)消息
![2-channels-subs](image/2-cnannels-subs.png "Channels-Subscription Mode")

* 第一步创建Source,将事件发送到Channel
```
kubectl apply -f lab3/11-eventing-source.yaml
```
* 第二步创建Channel 用来保存事件
```
# Channel可以是memoryChannel，也可以是kafkaChannel，这里我们定义一个memoryChannel
kubectl apply -f lab3/12-eventing-channel.yaml 
```
* 第三步创建服务 
```
kubectl apply -f lab3/13-eventing-helloa-service.yaml
kubectl apply -f lab3/13-eventing-hellob-service.yaml
```
* 第四步 创建Subscription 定与Channel中的消息
```
kubectl apply -f lab3/14-eventing-helloa-sub.yaml
kubectl apply -f lab3/14-eventing-hellob-sub.yaml
```

* 第五步 
```
#观察行为
stern eventinghello -c user-container
```
* 清理
```
...
k delete -f lab3/12-eventing-channel.yaml
k delete -f lab3/14-eventing-helloa-sub.yaml 
k delete -f lab3/14-eventing-hellob-sub.yaml 

```

## 模式3 使用Broker 基于元数据分发消息
![2-broker-trigger](image/3-broker-trigger.svg "Broker Trigger Mode")

knative 支持创建Broker 实现简单的基于CloudEvent元数据的内容分发路由
* 第一步 激活eventing的默认路由
```
#这个是对namespace域起作用的
kubectl label ns serverlesslab3 knative-eventing-injection=enabled
#执行完了之后，可以看到有一个默认的broker创建,并且会创建两个pod: default-broker-filter+ default-broker-ingress
kubectl get broker

#expose 外部route 这样我们可以给broker发 cloudevent用来测试
oc expose svc default-broker

```
* 第二步 部署两个ksvc
```
kubectl apply -f lab3/21-eventing-aloha-service.yaml
kubectl apply -f lab3/21-eventing-bonjour-service.yaml
```

* 第三步  部署Trigger
Eventing Trigger 就是用来定义 过滤条件的，它的Controller已经在第一步部署上了，就是default-broker-filter
```
kubectl apply -f lab3/22-trigger-helloaloha.yaml 
kubectl apply -f lab3/22-trigger-hellobonjour.yaml
```
* 第四步 发送CloudEvent
```
#ROUTE_HELLOA
#ROUTE_HELLOB
#ROUTE_BROKER
#测试1 直接给HelloA 发送http请求
curl -v "${ROUTE_HELLOA}" \
-X POST \
-H "Ce-Id: say-hello" \
-H "Ce-Specversion: 1.0" \
-H "Ce-Type: aloha" \
-H "Ce-Source: mycurl" \
-H "Content-Type: application/json" \
-d '{"message":"from a curl"}'

#测试2 直接给HelloB 发送http 请求
curl -v "${ROUTE_HELLOB}" \
-X POST \
-H "Ce-Id: say-hello" \
-H "Ce-Specversion: 1.0" \
-H "Ce-Type: bonjour" \
-H "Ce-Source: mycurl" \
-H "Content-Type: application/json" \
-d '{"message":"from a curl"}'

#测试3 给default-broker发送一个CloudEvent
curl -v "${ROUTE_BROKER}" \
-X POST \
-H "Ce-Id: say-hello" \
-H "Ce-Specversion: 1.0" \
-H "Ce-Type: greeting" \
-H "Ce-Source: mycurl" \
-H "Content-Type: application/json" \
-d '{"message":"from a curl"}'

```

清理
```
for i in lab3/2*.yaml;do kubectl delete -f $i; done
```
# 实验 4 knative evening II(使用kafka存储消息)
上面的实验，我们已经练习了如何使用Channel, Subscription, Broker, Trigger来控制事件的存储，订阅，分发. 这基本上已经覆盖了当前knative eventing 的功能。 

但是在生产中，为了消息保存的健壮性和持久性， 我们往往希望使用kafka来存储消息。
下面我们要使用knative的扩展功能 kafkaSource

创建一个新的命名空间
```
kubectl create ns serverlesslab4
```


**注意事项
请检查 你当前的namespace是否是 serverlesslab4, 如果使用不同的名称，需要改lab4里面的脚本**

## 安装kafkaSource
* 第一步 安装strimiz + knative kafkachannel组件
如果已经安装了，请忽略这个步骤
```
kubectl create ns kafka
kubectl apply -f lab4/installkafka/kafka-source.yaml
kubectl apply -f lab4/installkafka/kafka-channel.yaml
```
* 第二步 检查kafkaSource是否安装好
```
kubectl api-resources --api-group='sources.knative.dev'
kubectl api-resources --api-group='sources.knative.dev'
NAME               SHORTNAMES   APIGROUP              NAMESPACED   KIND
apiserversources                sources.knative.dev   true         ApiServerSource
camelsources                    sources.knative.dev   true         CamelSource
kafkasources                    sources.knative.dev   true         KafkaSource
pingsources                     sources.knative.dev   true         PingSource
sinkbindings                    sources.knative.dev   true         SinkBinding

kubectl api-resources --api-group='messaging.knative.dev'
NAME               SHORTNAMES   APIGROUP                NAMESPACED   KIND
channels           ch           messaging.knative.dev   true         Channel
inmemorychannels   imc          messaging.knative.dev   true         InMemoryChannel
kafkachannels      kc           messaging.knative.dev   true         KafkaChannel
subscriptions      sub          messaging.knative.dev   true         Subscription
```
* 第二步 把当前namespace配置为 默认使用kafkaChannel为knative Channel
```
kubectl apply -f lab4/01-default-channel-config.yaml
```
* 第三步 测试创建一个knative channel
```
kubectl apply -f lab4/02-my-events-channel
```
* 第四步 验证 kafkaChannel已经创建
```
kubectl get channels
kubectl get kafkachannels
bin/kafka-list-topics
# 删除调刚才创建的my-event-ch
kubectl delete channels my-events-ch
bin/kafka-list-topics
```
## 使用kafka进行事件转发与订阅
* 第一步 创建ksvc
```
kubectl apply -f lab4/11-eventing-hello-service.yaml
```
* 第二步 创建一个kafkaTopic
```
kubectl apply -f lab4/12-kafka-topic-my-topic.yaml
```
* 第三步 创建一个KafkaSource 并且直接链接(Sink)到新建的服务上
```
kubectl apply -f lab4/13-mykafka-source.yaml
```
* 第四步 测试
连接到kafkaTopic: my-topic上面，发送数据，然后观察eventing hello服务被调起

打开两个三个终端，分别命令:

```
#Terminal1
stern eventinghello -c user-container 
>观察eventinghello k服务被调起 并且显示 POST: Terminal2中键入的消息
```

```
#Terminal2
lab4/bin/kafka-producer.sh 
>然后键入消息
```
```
#Terminal3
lab4/bin/kafka-consumer.sh 
>监控my-topic中的消息
```




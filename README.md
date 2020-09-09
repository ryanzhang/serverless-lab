# 试验环境准备
Openshift 4.4 (kubernets 1.17) 

https://console-openshift-console.apps.ocp4-beijing-df27-ipi.azure.opentlc.com

xkJRn-qKK7b-33LJd-6n9Cj

客户端 for linux: 
| Command | 安装 |
| --- | --- |
| oc/kubectl| https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.4.3/openshift-client-linux-4.4.3.tar.gz
|kn| https://github.com/knative/client/releases/tag/v0.14.0
|http| dnf install httpie
|hey|https://storage.googleapis.com/jblabs/dist/hey_linux_v0.1.2
|stern|https://github.com/wercker/stern/releases/download/1.6.0/stern_linux_amd64

# 试验 1 knative serving 自动伸缩试验
## 根据Request 进行伸缩

## 根据CPU metrics进行伸缩
## 流量切换

# 试验 2 knative pipeline
## 从源代码编译 到 serverless部署
## Git push 自动触发 pipeline 运行

# 试验 3 knative eventing I
## 模式1 source 直接触发 Service 
## 模式2 使用Channel保存和群发(订阅)消息
## 模式3 使用Broker 基于元数据分发消息

# 试验 4 knative evening II(使用kafka存储消息)
## 使用kafka进行事件保存与订阅


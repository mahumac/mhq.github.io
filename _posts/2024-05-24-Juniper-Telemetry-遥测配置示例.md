---
title: Juniper Telemetry 遥测配置示例
date: 2024-05-24 +0800
categories: [网络]
description: 
pin: false
tags: [Prometheus, Juniper, Telemetry, 遥测] 
---

使用 telegraf 为 gRPC（Junos 遥测接口或 JTI）和 gNMI 配置遥测的示例。

参考文档：

​	[Telemetry configuration example](https://supportportal.juniper.net/s/article/telemetry-configuration-example?language=en_US)

[	Junos Telemetry Interface User Guide | Junos OS | Juniper Networks](https://www.juniper.net/documentation/us/en/software/junos/interfaces-telemetry/index.html)

使用 telegraf 为 gRPC（Junos 遥测接口或 JTI）和 gNMI 配置遥测的示例。

## 配置方法

### 配置 gRPC 服务 (telemetry sensor) 和 SSH：

```bash
set system services ssh
set system services extension-service request-response grpc clear-text address 10.0.0.2
set system services extension-service request-response grpc clear-text port 32767
set system services extension-service request-response grpc skip-authentication
set system services extension-service notification allow-clients address 10.0.0.1/32
```

### 配置 gRPC 服务防火墙策略

```bash
set policy-options prefix-list pl_JTI_server 10.0.0.1/32
set firewall family inet filter ACCEPT-JTI term ACCEPT-JTI from source-prefix-list pl_JTI_server
set firewall family inet filter ACCEPT-JTI term ACCEPT-JTI from protocol tcp
set firewall family inet filter ACCEPT-JTI term ACCEPT-JTI from port 32768
set firewall family inet filter ACCEPT-JTI term ACCEPT-JTI then count jti-count
set firewall family inet filter ACCEPT-JTI term ACCEPT-JTI then accept
```

### 配置安装 Telegraf ：

```bash
echo "deb https://repos.influxdata.com/${DISTRIB_ID,,} ${DISTRIB_CODENAME} stable" | sudo tee /etc/apt/sources.list.d/influxdb.list
# sudo apt-get install influxdb telegraf
# sudo service influxdb start
```

这是 Telegraf 的配置文件，位于 /etc/telegraf/telegraf.conf

可以使用两种方法，二选一。

#### 方法一： 使用 inputs.jti_openconfig_telemetry

这个插件使用 [Junos Telemetry Interface (JTI)](https://www.juniper.net/documentation/en_US/junos/topics/concept/junos-telemetry-interface-oveview.html)，读取Juniper 的OpenConfig遥测数据的实现。

```toml
# inputs.jti_openconfig_telemetry
[agent]
  interval = "10s"
  round_interval = true
  metric_batch_size = 1000
  metric_buffer_limit = 10000
  collection_jitter = "0s"
  flush_interval = "10s"
  flush_jitter = "0s"
  precision = "0s"
  hostname = ""

[[inputs.jti_openconfig_telemetry]]
  servers = ["10.0.0.2:32767"] <<< Use switch IP.

  username = "root" 
  password = "Juniper"
  client_id = "Switch"
  tags = {location = "HKG"}        # <<< 自定义tags

  sample_frequency = "2000ms"      # <<< 订阅上报频率，Juniper建议最小值为2s，根据实际情况调整
  sensors = [
    "/junos/system/linecard/cpu/memory",
    "/junos/system/linecard/firewall/",
    "/junos/system/linecard/interface/",
    "/junos/system/linecard/interface/logical/usage",
    "/junos/system/linecard/packet/usage/",
    "/network-instances/network-instance[instance-name='name']/protocols/protocol/evpn/irb-interfaces/",
    "/interfaces/interface/state/",
    "/junos/system/linecard/environment",
    "/junos/system/linecard/optics/",
  ]
  collection_jitter = "0s"
  flush_interval = "15s"
  flush_jitter = "0s"

  fielddrop = ["/interfaces/interface/subinterfaces/subinterface/ipv6/neighbors/neighbor/state/is-router"]

[[processors.regex]]
  # 将匹配的文本替换为指定的文本
  # 如果"/interfaces/interface/subinterfaces/subinterface/state/description"字段的值为空，将这个字段的值替换为 “-”
  [[processors.regex.fields]]
    key = "/interfaces/interface/subinterfaces/subinterface/state/description"
    pattern = "^$"
    replacement = "-"
    
[[outputs.prometheus_client]]
  listen = ":9273"
  path = "/metrics"
  
  # Expiration interval for each metric. 0 == no expiration
  expiration_interval = "5s"
  export_timestamp = true  # 将timestamp 附加到metrics, 让 prometheus以该时间戳为准（不使用prometheus的job scrape 时间戳）
  metric_buffer_limit = 100000

  [outputs.prometheus_client.tagpass]
    location = ["HKG"]
```

#### 方法二： 使用 inputs.gnmi

这个插件使用 [gNMI](https://github.com/openconfig/reference/blob/master/rpc/gnmi/gnmi-specification.md) Subscribe 方法的遥测数据。这个输入插件是 与供应商无关，并且在支持 gNMI 规范的任何平台上都受支持。

参考：[telegraf/plugins/inputs/gnmi at master · influxdata/telegraf](https://github.com/influxdata/telegraf/tree/master/plugins/inputs/gnmi)

 ```toml
 # inputs.gnmi
 [agent]
   interval = "10s"
   round_interval = true
   metric_batch_size = 1000
   metric_buffer_limit = 10000
   collection_jitter = "0s"
   flush_interval = "10s"
   flush_jitter = "0s"
   precision = "0s"
   hostname = ""
 
 [[outputs.prometheus_client]]
    listen = ":9273"
    path = "/metrics"
 
 [[inputs.gnmi]]
    addresses = ["10.0.0.2:32767"]    # <<< Use switch IP.
 
    [[inputs.gnmi.subscription]]
       name = "ifcounters"
       origin = "openconfig-interfaces"
       path = "/interfaces/interface/state/counters"
       subscription_mode = "sample"
       sample_interval = "10s"
 
    [[inputs.gnmi.subscription]]
       name = "component"
       origin = "openconfig-system"
       path = "/components/component/state/name"
       subscription_mode = "sample"
       sample_interval = "10s"
 ```

一旦配置完成，就可以配置 VMX 中的接口并建立可访问性，用以下命令启动 Telegraf 服务:

```bash
root@ubuntu:/etc/telegraf# systemctl restart telegraf
root@ubuntu:/etc/telegraf# systemctl status telegraf
● telegraf.service - Telegraf
   Loaded: loaded (/lib/systemd/system/telegraf.service; enabled; vendor preset: enabled)
   Active: active (running) since Wed 2023-04-12 10:00:41 PDT; 5s ago
    Docs: https://github.com/influxdata/telegraf
  Main PID: 80274 (telegraf)
   Tasks: 7 (limit: 2237)
   Memory: 21.5M
    CPU: 159ms
   CGroup: /system.slice/telegraf.service
       └─80274 /usr/bin/telegraf -config /etc/telegraf/telegraf.conf -config-directory /etc/telegraf/telegraf.d
Apr 12 10:00:41 ubuntu telegraf[80274]: 2023-04-12T17:00:41Z I! Available plugins: 235 inputs, 9 aggregators, 27 processors, 22 parsers, 57 outputs, >
Apr 12 10:00:41 ubuntu telegraf[80274]: 2023-04-12T17:00:41Z I! Loaded inputs: jti_openconfig_telemetry
Apr 12 10:00:41 ubuntu telegraf[80274]: 2023-04-12T17:00:41Z I! Loaded aggregators:
Apr 12 10:00:41 ubuntu telegraf[80274]: 2023-04-12T17:00:41Z I! Loaded processors:
Apr 12 10:00:41 ubuntu telegraf[80274]: 2023-04-12T17:00:41Z I! Loaded secretstores:
Apr 12 10:00:41 ubuntu telegraf[80274]: 2023-04-12T17:00:41Z I! Loaded outputs: prometheus_client
Apr 12 10:00:41 ubuntu telegraf[80274]: 2023-04-12T17:00:41Z I! Tags enabled: host=ubuntu
Apr 12 10:00:41 ubuntu systemd[1]: Started Telegraf.
Apr 12 10:00:41 ubuntu telegraf[80274]: 2023-04-12T17:00:41Z I! [agent] Config: Interval:10s, Quiet:false, Hostname:"ubuntu", Flush Interval:10s
Apr 12 10:00:41 ubuntu telegraf[80274]: 2023-04-12T17:00:41Z I! [outputs.prometheus_client] Listening on http://[::]:9273/metrics
```

查看sensors 的状态: 

```bash
root@ubuntu:/etc/telegraf# curl localhost:9273/metrics | grep sensors
```

## Prometheus配置

```yaml
  - job_name: "JTI-HK1-1"
    scrape_interval: 1s
    honor_timestamps: true		# 以metric自带的时间戳为准
    honor_labels: true
    static_configs:
      - targets: ["localhost:9273"]
```

##  Grafana 查询接口流量

 查询单个物理接口 xe-0/0/0:2, index=任意

```bash
# 查询单个物理接口 xe-0/0/0:2, index=任意
rate(
    interfaces__interfaces_interface_subinterfaces_subinterface_state_counters_in_octets{
        interfaces_interface__name=~"xe-0/0/0:2",
        interfaces_interface_subinterfaces_subinterface__index=~".+",
        system_id=~"HKG02.+"
    } [$__rate_interval]
) *8
+ on (system_id, interfaces_interface__name, interfaces_interface_subinterfaces_subinterface__index) 
group_left (interfaces_interface_subinterfaces_subinterface_state_description)
last_over_time(
    interfaces__interfaces_interface_subinterfaces_subinterface_state_index{
        interfaces_interface_subinterfaces_subinterface_state_oper_status="UP",
        interfaces_interface__name=~"xe-0/0/0:2",
        interfaces_interface_subinterfaces_subinterface__index=~".+",
        interfaces_interface_subinterfaces_subinterface_state_description=~".+",
        system_id=~"HKG02.+"
    } [2m]
) *0
```

查询单个物理接口 et-0/1/5, 指定index， 例如：index=~"536|538|541|552|568"

```bash
# 查询单个物理接口 et-0/1/5, index=~"536|538|541|552|568"
-rate(
    interfaces__interfaces_interface_subinterfaces_subinterface_state_counters_out_octets{
        interfaces_interface__name=~"et-0/1/5",
        interfaces_interface_subinterfaces_subinterface__index=~"536|538|541|552|568",
        system_id=~"HKG02.+"
    } [$__rate_interval]
) *8
+ on (system_id, interfaces_interface__name, interfaces_interface_subinterfaces_subinterface__index) 
group_left (interfaces_interface_subinterfaces_subinterface_state_description)
last_over_time(
    interfaces__interfaces_interface_subinterfaces_subinterface_state_index{
        interfaces_interface_subinterfaces_subinterface_state_oper_status="UP",
        interfaces_interface__name=~"et-0/1/5",
        interfaces_interface_subinterfaces_subinterface__index=~"536|538|541|552|568",
        interfaces_interface_subinterfaces_subinterface_state_description=~".+",
        system_id=~"HKG02.+"
    } [2m]
) *0
```

查询聚合接口 ae3.0，仅显示成员接口的速率(不显示聚合接口ae3.0)

```bash
# 查询聚合接口ae3.0，仅显示成员接口的速率(不显示聚合接口ae3.0)
    rate(
        interfaces__interfaces_interface_subinterfaces_subinterface_state_counters_${direction}_octets{
            interfaces_interface_subinterfaces_subinterface_state_parent_ae_name=~"ae3.0",
            interfaces_interface_subinterfaces_subinterface__index=~"0",
            system_id=~"SJC03.+"
        } [$__rate_interval]
    ) *8
    + on (system_id, interfaces_interface_subinterfaces_subinterface__index) 
    group_left (interfaces_interface_subinterfaces_subinterface_state_description)
    last_over_time(
        interfaces__interfaces_interface_subinterfaces_subinterface_state_index{
            interfaces_interface_subinterfaces_subinterface_state_oper_status="UP",
            interfaces_interface__name=~"ae3",
            interfaces_interface_subinterfaces_subinterface_state_name=~"ae3.0",
            interfaces_interface_subinterfaces_subinterface__index=~"0",
            #interfaces_interface_subinterfaces_subinterface_state_description=~".+",
            system_id=~"SJC03.+"
        } [2m]
    ) *0
```

查询聚合接口ae14.1，将所有成员接口速率相加（使用sum函数），最终显示聚合接口ae14.1的速率

```bash
# 查询聚合接口ae14.1，将所有成员接口速率相加（使用sum函数），最终显示聚合接口ae14.1的速率
sum(
    rate(
        interfaces__interfaces_interface_subinterfaces_subinterface_state_counters_out_octets{
            interfaces_interface_subinterfaces_subinterface_state_parent_ae_name=~"ae14.1",
            #interfaces_interface_subinterfaces_subinterface__index=~".+",
            system_id=~"SJC04.+"
        } [$__rate_interval]
    ) *8
    + on (system_id, interfaces_interface_subinterfaces_subinterface__index) 
    group_left (interfaces_interface_subinterfaces_subinterface_state_description)
    last_over_time(
        interfaces__interfaces_interface_subinterfaces_subinterface_state_index{
            interfaces_interface_subinterfaces_subinterface_state_oper_status="UP",
            interfaces_interface__name=~"ae14",
            interfaces_interface_subinterfaces_subinterface_state_name=~"ae14.1",
            #interfaces_interface_subinterfaces_subinterface__index=~".+",
            #interfaces_interface_subinterfaces_subinterface_state_description=~".+",
            system_id=~"SJC04.+"
        } [2m]
    ) *0
)
by (
    system_id,
    interfaces_interface_subinterfaces_subinterface_state_parent_ae_name,
    interfaces_interface_subinterfaces_subinterface_state_description
)
```

显示 options - Legend

```
{{system_id}}: {{interfaces_interface_subinterfaces_subinterface_state_parent_ae_name}} ({{interfaces_interface__name}}) - out- {{interfaces_interface_subinterfaces_subinterface_state_description}}
```



相关信息

[https://github.com/Juniper/yang/tree/master/23.4/23.4R1.10/openconfig/models
https://github.com/influxdata/telegraf/blob/master/plugins/inputs/jti_openconfig_telemetry/README.md
https://github.com/influxdata/telegraf/blob/master/plugins/inputs/gnmi/README.md](https://github.com/Juniper/yang/tree/master/23.4/23.4R1.10/openconfig/models)[telegraf/plugins/inputs/gnmi at master · influxdata/telegraf](https://github.com/influxdata/telegraf/tree/master/plugins/inputs/gnmi)
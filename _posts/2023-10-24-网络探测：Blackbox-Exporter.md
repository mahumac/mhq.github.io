---
title: 网络探测：Blackbox Exporter
date: 2023-10-24 +0800
categories: [Prometheus, Blackbox]
pin: false
math: true
tags: [Prometheus, Blackbox] 
---

Blackbox Exporter是Prometheus社区提供的官方黑盒监控解决方案，其允许用户通过：HTTP、HTTPS、DNS、TCP以及ICMP的方式对网络进行探测。

# 下载安装 Blackbox Exporter

```bash
cd ~
wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.25.0/blackbox_exporter-0.25.0.linux-amd64.tar.gz
```

```bash
mkdir /etc/blackbox_exporter/
tar -xvf  blackbox_exporter-0.25.0.linux-amd64.tar.gz
cp blackbox_exporter-0.25.0.linux-amd64/blackbox_exporter  /usr/local/bin/
cp blackbox_exporter-0.25.0.linux-amd64/blackbox.yml  /etc/blackbox_exporter/
```

设置systemd 启动服务

```bash
cat > /etc/systemd/system/blackbox_exporter.service << EOF
[Unit]
Description= Prometheus blackbox_exporter
Documentation=https://prometheus.io/
After=network.target

[Service]
Type=simple
User=nobody
ExecStart=/usr/local/bin/blackbox_exporter --config.file=/etc/blackbox_exporter/blackbox.yml
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF
```

设置防火墙，并启动 blackbox_exporte

```bash
systemctl enable --now blackbox_exporter
firewall-cmd --add-port=9115/tcp --permanent
firewall-cmd --reload
```

## ICMP 权限

ICMP 探测需要提升的权限才能运行：

- *Windows*：需要管理员权限。

- *Linux*：需要具有`cap_net_raw`用户组权限 或者 root 权限 。执行以下命令，修改`net.ipv4.ping_group_range`

  ```bash
  sysctl  net.ipv4.ping_group_range = 0  2147483647
  ```

# ICMP 监控

## blackbox.yml 配置

```yaml
modules:
...  ...
  icmp:
    prober: icmp
    icmp:
      preferred_ip_protocol: ip4  # <<< 优先IPv4
  icmp_ttl5:
    prober: icmp
    timeout: 5s
    icmp:
      ttl: 5
```

# 与Prometheus集成

假设有三个机房，分别位于GZ、SH、BJ。在每个机房都各部署一台blackbox_exporter 探针（Prober）:

```tex
                               -----------
                        +--   | Prober-BJ |   --+
                        |      -----------      |
                        |                       |
 -------------------    |      -----------      |         --------- 
| prometheus server | --+--   | Prober-GZ |   --+---  ---| Targets |
 -------------------    |      -----------      |         --------- 
                        |                       |
                        |      -----------      |
                        +--   | Prober-SZ |   --+
                               -----------
```



```yaml
# Prometheus 配置
scrape_configs:
  - job_name: "blackbox_icmp-01"
    scrape_interval: 1s
    metrics_path: /probe
    params:
      module: [icmp]
    file_sd_configs:
      - files: ["/etc/prometheus/file_sd_config.d/blackbox_icmp_targets.yml"]
        refresh_interval: 1m
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - source_labels: [__param_module]    # 将内置标签`__param_module`修改为`module`
        target_label: module
      - target_label: __address__
        replacement: 127.0.0.1:9115
      - target_label: prober               # 自定义标签,GZ机房
        replacement: GZ
      - target_label: prober_ip
        replacement: 1.1.1.1               # GZ机房探针公网ip地址

  - job_name: "blackbox_icmp-02"
    scrape_interval: 1s
    metrics_path: /probe
    params:
      module: [icmp]
    file_sd_configs:
      - files: ["/etc/prometheus/file_sd_config.d/blackbox_icmp_targets.yml"]
        refresh_interval: 1m
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - source_labels: [__param_module]     # 将内置标签`__param_module`修改为`module`
        target_label: module
      - target_label: __address__
        replacement: 2.2.2.2:9115
      - target_label: prober                # 自定义标签，SH机房
        replacement: SH
      - target_label: prober_ip             
        replacement: 2.2.2.2                # SH机房探针ip地址

  - job_name: "blackbox_icmp-03"
    scrape_interval: 1s
    metrics_path: /probe
    params:
      module: [icmp]
    file_sd_configs:
      - files: ["/etc/prometheus/file_sd_config.d/blackbox_icmp_targets.yml"]
        refresh_interval: 1m
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - source_labels: [__param_module]     # 将内置标签`__param_module`修改为`module`
        target_label: module
      - target_label: __address__
        replacement: 3.3.3.3:9115
      - target_label: prober                # 自定义标签,BJ机房
        replacement: BJ
      - target_label: prober_ip             
        replacement: 3.3.3.3                # BJ机房探针ip地址
```

## blackbox_icmp_targets.yml 文件配置

```yaml
##################
# targets-TYO
- labels:
    idc: TYO
    profile: CN2
  targets:
    - 154.82.64.2
    - 154.82.65.3
##################
# targets-TPE
- labels:
    idc: TPE
    profile: CN2
  targets:
    - 156.248.76.3
- labels:
    idc: TPE
    profile: 回国优化
  targets:
    - 156.248.75.2
    - 156.248.77.3
##################
# targets-HKG
- labels:
    idc: HK
    profile: CN2
  targets:
    - 154.82.72.2
    - 154.82.73.3
- labels:
    idc: HK-CMI
    profile: 回国优化
  targets:
    - 154.91.86.2
    - 154.91.86.3
```

## 查询 ICMP 丢包

### 使用PromQL 计算 ICMP 丢包

* 通过metric   `probe_success`查询 icmp ping 是否成功 (success = 1 , faile = 0)

```
probe_success{job=~"blackbox_icmp.*"}
```

* 通过以下以下PromQL，计算 60 秒内的 icmp ping 平均丢包率

```bash
1- avg_over_time(probe_success{job=~"blackbox_icmp.*",instance=~".+"}[60S])
```



###  使用PromQL 计算 ICMP RTT

* 通过以下以下PromQL, 计算60秒内的 平均 icmp rtt 值。

**注意：**如果出现 icmp 丢包，blackbox 会返回：

```tex
probe_success{} = 0                 # 其值为 0 说明探测失败，这个是预期的正确结果
probe_icmp_duration_seconds{} = 0   # icmp探测失败(即发生了丢包)，预期的结果应该是空值 或者一个无限大的值。这里返回值为`0`,不是预期的结果。
```

比方说，发送了5个 icmp request，中间出现2次丢包，收到了3个 icmp reply :

````bash
# * 代表出现丢包
11ms  *   *  9ms  10ms
````

如果直接使用 `avg_over_time ( probe_icmp_duration_seconds {} )[5s]` 来计算5秒内的 平均 ICMP RTT，结果如下：
$$
(11 + 0 + 0 + 9 + 10) / 5 = 30 / 5 = 6  毫秒
$$
明显得到一个错误的结果。预期的平均 RTT  应该是：
$$
(10 + 9 + 10) / 3 = 30 / 3 = 10  毫秒
$$
需要排除 `probe_icmp_duration_seconds{} = 0` 的 情况。

应该使用以下PromQL 公式：

```bash
# 排除probe_icmp_duration_seconds{}=0的情况, 使用内部子查询（Subquery，1s精度）
avg_over_time(
    ( probe_icmp_duration_seconds{
         job=~"blackbox_icmp.*", phase="rtt"
      } > 0
	) [$interval:1s]
)

# 或者：
sum_over_time(
    probe_icmp_duration_seconds{
        job=~"blackbox_icmp.*", phase="rtt"
    } [$interval]
) 
/ ignoring(phase) 
sum_over_time(probe_success{job=~"blackbox_icmp.*"}[$interval])
```

### 计算 ICMP Jitter

```bash
# 计算 ICMP抖动 > 15ms 的IP
# 排除probe_icmp_duration_seconds{}=0的情况, 使用内部子查询（Subquery，1s精度）
stddev_over_time(
	( probe_icmp_duration_seconds{
    	job=~"blackbox_icmp.*", phase="rtt"
      } > 0
     ) [$interval:1s]
 ) > 0.015
```

### 使用 Record 规则 计算 ICMP 丢包

也可以使用 promtheus 记录规则来计算 ICMP 丢包情况：

```yaml
#  prometheus 配置
rule_files:
  - "/etc/prometheus/rule_files.d/*.yml"
```

rule_files文件配置：

```yaml
groups:
  - name: ICMP Reachability
    interval: 1s
    rules:
    # icmp loss 丢包率
    - record: blackbox_icmp:probe_success:avg_over_time
      expr: avg_over_time(probe_success{job=~"blackbox_icmp.*"}[60s])

    # 60秒内 最小 rtt , 使用Subquery表达式（1s精度），排除掉丢包的情况（排除probe_success{} = 0）
    - record: blackbox_icmp:probe_icmp_duration_seconds:min_over_time
      expr: min_over_time( (probe_icmp_duration_seconds{job=~"blackbox_icmp.*", phase="rtt"}>0) [60s:1s])
    # 60秒内 最大 rtt
    - record: blackbox_icmp:probe_icmp_duration_seconds:min_over_time
      expr: min_over_time(probe_icmp_duration_seconds{job=~"blackbox_icmp.*", phase="rtt"}[60s])

    # 直方图,使用Subquery表达式（1s精度），排除掉丢包的情况（排除probe_success{} = 0）
    - record: blackbox_icmp:probe_icmp_duration_seconds:quantile_over_time_25
      expr: quantile_over_time(0.25, (probe_icmp_duration_seconds{job=~"blackbox_icmp.*", phase="rtt"}>0) [60s:1s])
    - record: blackbox_icmp:probe_icmp_duration_seconds:quantile_over_time_50
      expr: quantile_over_time(0.5,  (probe_icmp_duration_seconds{job=~"blackbox_icmp.*", phase="rtt"}>0) [60s:1s])
    - record: blackbox_icmp:probe_icmp_duration_seconds:quantile_over_time_75
      expr: quantile_over_time(0.75, (probe_icmp_duration_seconds{job=~"blackbox_icmp.*", phase="rtt"}>0) [60s:1s])
    - record: blackbox_icmp:probe_icmp_duration_seconds:quantile_over_time_75
      expr: quantile_over_time(0.95, (probe_icmp_duration_seconds{job=~"blackbox_icmp.*", phase="rtt"}>0) [60s:1s])

  - name: ICMP Loss Alert
    interval: 30s
    rules:
    - alert: PacketLoss
      expr: (1 - avg_over_time(blackbox_icmp:probe_success:avg_over_time[5m])) * 100 > 10
      labels:
      annotations:
        description: "ICMP packet loss for {{ $labels.instance }} - has been over 10% ({{ $value }}%) for 5 minutes."
        summary: "ICMP packet loss over 10%"
```

-----------------------

# 使用单个 Job 访问多个模块和目标

以 http、https 模块为例

1） 将 Blackbox Exporter job添加到`prometheus.yml`文件中：

```yaml
 scrape_configs:
  # Blackbox Exporter
  - job_name: 'blackbox'
    scrape_interval: 10s
    metrics_path: /probe
    file_sd_configs:
      - files:
          - '/etc/prometheus/blackbox/targets/blackbox_exporter-http*.yml'
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [module]         # 将 blackbox_exporter-http*.yml 文件中的 `module`标签修改为 `__param_module`
        target_label: __param_module
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 127.0.0.1:9115     # blackbox exporter
 # End Blackbox Exporter
```

2）然后需要配置 Blackbox Exporter 如何处理我们的两个配置文件（一个用于http，一个用于https）：

```yaml
# Blackbox.yml
modules:
  https_2xx:
    prober: http
    timeout: 5s
    http:
      method: GET
      no_follow_redirects: false
      fail_if_ssl: false
      fail_if_not_ssl: true
      preferred_ip_protocol: "ipv4"
  http_2xx:
    prober: http
    timeout: 5s
    http:
      method: GET
      no_follow_redirects: false
      fail_if_ssl: true
      fail_if_not_ssl: false
      preferred_ip_protocol: "ipv4"
```

3）最终的targets文件配置。这些是黑匣子导出器将监视的站点的实际列表。

```yaml
# nano /etc/prometheus/blackbox/targets/*.yml
- labels:
    module: https_2xx         # https, 对应 prometheus.yml文件中 relabel_configs配置块中的 source_labels: [module]
  targets:
  - https://www.google.com
 
- labels:
    module: http_2xx          # http
  targets:
  - http://www.baidu.com/
```

# 未完待续

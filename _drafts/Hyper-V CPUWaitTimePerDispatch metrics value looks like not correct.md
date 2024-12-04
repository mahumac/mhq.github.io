# Hyper-V CPUWaitTimePerDispatch metrics value looks like not correct

### Environment

- windows_exporter版本：0.29.2
- Windows Server 版本：2025

### 问题陈述

```bash
# HELP windows_hyperv_vm_cpu_wait_time_per_dispatch_total Time in nanoseconds waiting for a virtual processor to be dispatched onto a logical processor
# TYPE windows_hyperv_vm_cpu_wait_time_per_dispatch_total counter
windows_hyperv_vm_cpu_wait_time_per_dispatch_total{core="0",vm="Prometheus"} 6.65257352e+09
windows_hyperv_vm_cpu_wait_time_per_dispatch_total{core="1",vm="Prometheus"} 5.835522719e+09
windows_hyperv_vm_cpu_wait_time_per_dispatch_total{core="2",vm="Prometheus"} 5.591472665e+09
windows_hyperv_vm_cpu_wait_time_per_dispatch_total{core="3",vm="Prometheus"} 5.293499635e+09
```

可以看到获取到的 metric `windows_hyperv_vm_cpu_wait_time_per_dispatch_total`类型是 `counter`,

`windows_hyperv_vm_cpu_wait_time_per_dispatch_total{core="0",vm="Prometheus"} `的值为`6.65257352e+09`, 换算成十进制数为 6652573520 ns ，约等于 66.53 μs

在granfan中查看：

![image-20241129201027164](./Hyper-V CPUWaitTimePerDispatch metrics value looks like not correct.assets/image-20241129201027164-1732882229103-3.png)

`6652573520`这个值找不到，但是可以发现这个值 位于 `20:02:40 `-- `20:02:50`时间段之间。

![image-20241129201214297](./Hyper-V CPUWaitTimePerDispatch metrics value looks like not correct.assets/image-20241129201214297-1732882335820-5.png)

在 time series 图中，也可以看到 `windows_hyperv_vm_cpu_wait_time_per_dispatch_total{core="0",vm="Prometheus"}`的值是一个不断增长的counter类型。



但是在windows perfmon性能查看器中查看其实际值，显示如下：

![image-20241129202041740](./Hyper-V CPUWaitTimePerDispatch metrics value looks like not correct.assets/image-20241129202041740.png)

`windows_hyperv_vm_cpu_wait_time_per_dispatch_total{core="0",vm="Prometheus"}` 的值看上去应该是gauge类型。真实的值 11.94 μs ~ 58.98  μs 之间，平均值 大约等于 27.60 μs .
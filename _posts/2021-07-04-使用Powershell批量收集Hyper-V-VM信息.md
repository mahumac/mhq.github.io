---
title: 使用Powershell批量收集Hyper-V VM信息
date: 2021-07-04 12:12:12 +0800
categories: [Windows, Hyper-V]
description: 
tags: [Windows,Hyper-V,Powershell,VM] 
---

# Powershell 批量收集Hyper-V  VM信息

```powershell
############################################################################################
# 批量查询VM信息，并导出为csv文件
#    名称、计算机名、IP地址、内存、CPU、创建时间等等
#
# 注意 ：适用于SCVMM 控制台的Powershell命令，如果要用于本地Hyper-V主机，需要修改以下参数名 
#    VirtualNetworkAdapters ——> NetworkAdapters
#    VirtualHardDisks       ——> HardDisks
#    IPv4Addresses          ——> IPAddresses
#    Description            ——> Notes
############################################################################################

$allVM = Get-VM | Select-Object *
$vmlist = @()

foreach ( $VM in $allVM )
{
    $IP = Get-VM $VM.Name | Select-Object -ExpandProperty VirtualNetworkAdapters | Select-Object -ExpandProperty IPv4Addresses
    $DiskSize = [int]( (Get-VM $VM.Name | Select-Object -ExpandProperty VirtualHardDisks).TotalSize /1024/1024/1024)

    $VM | Add-Member -Membertype noteproperty -Name IP -value $IP 
    $VM | Add-Member -Membertype noteproperty -Name DiskSize -value $DiskSize
    $VMList+=$VM
}

$VMlist | Select name,ComputerName,IP,Description,CreationTime,OperatingSystem,CPUCount,Memory,DiskSize | ft 

$allVM = Get-VM | Select-Object *
$VMlist = @()

foreach ( $VM in $allVM )
{
    $IP = Get-VM $VM.Name | Select-Object -ExpandProperty VirtualNetworkAdapters | Select-Object -ExpandProperty IPv4Addresses
    $DiskSize = [int]( (Get-VM $VM.Name | Select-Object * ).TotalSize /1024/1024/1024)
    $VMObject = @()
    $VMObject = New-Object psobject

    $VMObject | Add-Member -Membertype noteproperty -Name 名称          -value $VM.Name 
    $VMObject | Add-Member -Membertype noteproperty -Name 计算机名      -value $VM.ComputerName
    $VMObject | Add-Member -Membertype noteproperty -Name IP地址        -value $IP
    $VMObject | Add-Member -Membertype noteproperty -Name 状态          -value $VM.Status
    $VMObject | Add-Member -Membertype noteproperty -Name CPU           -value $VM.CPUCount
    $VMObject | Add-Member -Membertype noteproperty -Name "内存(MB)"    -value $VM.Memory
    $VMObject | Add-Member -Membertype noteproperty -Name "磁盘(GB)"    -value $DiskSize
    $VMObject | Add-Member -Membertype noteproperty -Name 创建时间      -value $VM.CreationTime
    $VMObject | Add-Member -Membertype noteproperty -Name 描述信息      -value $VM.Description
    $VMObject | Add-Member -Membertype noteproperty -Name 操作系统      -value $VM.OperatingSystem

    $VMList+=$VMObject
}
$VMlist | Select-Object * | Format-Table
$VMlist | Export-Csv D:\VM_info.csv -Encoding UTF8 -Append -Force -ErrorAction:Continue
```

# 查询Guest VM的IP、MAC、vNic ID

```powershell
Get-VM DC01 | Select-Object -ExpandProperty VirtualNetworkAdapters | Select-Object Name,MacAddress,IPv4Addresses,ID
```

# 查询单台 VM 信息

```powershell
Get-vm | Select Name,ComputerName,Description,CreationTime,Generation,CPUCount,Memory,TotalSize | ft
```


---
title: IPBan安装配置（Windows）
date: 2023-01-10 +0800
categories: [Windows]
description: 
pin: false
tags: [Windows, Powershell, IPBan] 
---

# IPBan安装配置（Windows）

## 特征

- 通过检测事件查看器和/或日志文件中的失败登录来自动禁止 IP 地址。在 Linux 上，默认情况下会监视 SSH。在 Windows 上，会监视 RDP、OpenSSH、VNC、MySQL、SQL Server、Exchange、SmarterMail、MailEnable。可以通过配置文件轻松添加更多应用程序。
- 事件查看器和日志文件的其他方法如下：https://github.com/DigitalRuby/IPBan/tree/master/Recipes
- 高度可配置，许多选项可用于确定失败登录计数阈值、禁止时间等。
- 请务必查看 ipban.config 文件（以前称为 DigitalRuby.IPBan.dll.config，请参阅 IPBanCore 项目）以获取配置选项，每个选项都记录有注释。
- 事件查看器的禁令基本上是立即发生的。对于日志文件，您可以设置轮询更改的频率。
- 非常快 - 我从 2012 年开始优化和调整此代码。瓶颈几乎总是防火墙实现，而不是这段代码。
- 通过将 unban.txt 文件放入 service 文件夹，将每个 ip 地址放在一行上以解禁，轻松解禁 IP 地址。
- 适用于所有平台上的 IPv4 和 IPv6。
- 请访问 https://github.com/DigitalRuby/IPBan/wiki 的 wiki 以获取更多文档。



## 使用powershell一键脚本进行IPBAN的安装

参考1: [Windows 2012/2016系统远程访问安全配置以及安装IPBan的详细配置记录 - OMO萌](https://omo.moe/archives/968/)

参考2: [Home · DigitalRuby/IPBan Wiki --- 主页 · DigitalRuby/IPBan Wiki (github.com)](https://github.com/DigitalRuby/IPBan/wiki/Configuration)

IPBan的安装，主要是win2012环境需要配置强制NTLM v2访问，以及.net Frame work 4.5.2版本的安装 ，而win2016默认都ok，省略了这些手动配置

```powershell
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12; iex ((New-Object System.Net.WebClient).DownloadString('https://raw.githubusercontent.com/DigitalRuby/IPBan/master/IPBanCore/Windows/Scripts/install_latest.ps1'))
```

默认安装目录为：`C:\Program Files\IPBan`

### 1 基本配置

安装完毕后会自动配置相关服务，开启系统登录/注销事件安全审计，并且打开配置文件`ipban.conf`

**建议修改 `ipban.override.config`  , 使用此文件，可以覆盖 `ipban.config `文件中的条目，而无需修改原始配置**

ctrl+f 查找`appsettings`项，找到如下设置，一个是失败次数，另一个是封禁时间，默认是5次登录失败，封禁1天

```xml
<appSettings>
	<!-- 失败次数 before banning the ip address -->
	<add key="FailedLoginAttemptsBeforeBan" value="5"/>
    
    <!-- 封禁时长，DD:HH:MM:SS 格式。00:00:00:00 禁止 90 天（最长）-->
    <!-- 可以设置多个时间跨度,使用逗号分隔。必须是升序排列 -->
    <!-- 当一个被ban的ip地址BanTime到期后，再次出现failed login，就会将这个ip移动到下一个BanTime -->
    <add key="BanTime" value="00:00:60:00,01:00:00:00,30:00:00:00"/>
    
    <!-- 删除在此时间之后尚未被禁止的 failed logins 记录 -->
    <add key="ExpireTime" value="01:00:00:00"/>
... ...   
</appSettings>  
```

也可以手动访问`C:\Program Files\IPBan\ipban.override.config`文件随时进行调整

### 2 IPBan白名单

编辑配置文件`ipban.override.config`，以覆盖默认的ipban.config中的设置

-  **`Whitelist`** - 以逗号分隔的 IP 地址、URL 或 DNS 名称列表，这些列表永远不会被禁止。这些将被添加到防火墙的白名单规则中。如果您有静态 IP 地址，请将您的静态 IP 地址放在这里，这样您就不会不小心将自己锁在外面。

   ```xml
   <appSettings>
   	<add key="Whitelist" value="1.1.1.1, 2.2.2.2, 3.3.3.3.3"/>
   </appSettings>  
   ```

- **`WhitelistRegex`** - 用于白名单的正则表达式，允许模式匹配或 ip 地址范围。(http://www.regular-expressions.info/)

  ```xml
  <appSettings>
  	<add key="WhitelistRegex" value="^(172\.16\.0\..*)|(192\.168\.1\..*)|(10\.0\.0\..*)$"/>
  </appSettings> 
  ```

白名单优先于黑名单。如果您使用 url，则响应中的每个 ip 条目都应以换行符分隔，例如： https://uptimerobot.com/inc/files/ips/IPv4andIPv6.txt 。此配置条目中的 URL 应包含少量 IP 地址。

更改之后重启IPBan

```powershell
Get-Service IPBan | Restart-Service
```

### 3 强制封禁或解禁

将`ban.txt`文件放在与 IPBan 服务相同的文件夹中, 以便手动禁止 IP 地址。IP 地址为纯文本，每行一个 IP 地址。在下一个周期中，IPBan 将禁止文件中的每个 IP 地址，然后删除该文件。这对于只想引起 IP 地址禁令的外部应用程序非常有用，例如流量监视器、syn 洪水检测器等。

将`unban.txt`文件放在与 IPBan 服务相同的文件夹中, 来手动取消禁用 IP 地址。IP 地址为纯文本，每行一个 IP 地址。在下一个周期中，IPBan 将解禁文件中的任何 IP 地址，然后删除该文件。每个 IP 都将从防火墙和 IPBan 数据库中删除。

### 4 log设置

`nlog.confg`文件 输出的日志类别

```xml
<?xml version="1.0"?>
<nlog xmlns="http://www.nlog-project.org/schemas/NLog.xsd" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" throwExceptions="false" internalLogToConsole="false" internalLogToConsoleError="false" internalLogLevel="Trace">
  <targets>
    <target name="logfile" xsi:type="File" fileName="${basedir}/logfile.txt" archiveNumbering="Sequence" archiveEvery="Day" maxArchiveFiles="28" encoding="UTF-8"/>
    <target name="console" xsi:type="Console"/>
  </targets>
  <rules>
    <logger name="*" minlevel="Warn" writeTo="logfile"/>
    <logger name="*" minlevel="Info" writeTo="console"/>
  </rules>
</nlog>
```

- fileName="${basedir}/logfile.txt"   日志保存位置     
- maxArchiveFiles=28                          最多保留28份日志文件
- minlevel=Warn                                   日志级别高于info ,可用值"Debug, Info, Warn, Error, Critical"

### 5 Windows 下的 SSH配置

```powershell
 # 启用 OpenSSH.Server
 Get-WindowsCapability -Online -Name OpenSSH.Server* |Add-WindowsCapability -Online
 
 # 启用 sshd 服务
 Get-Service sshd | Set-Service -StartupType Automatic -Status Running -PassThru
```

Windows下的ssh配置文件位于`C:\ProgramData\ssh`目录（`C:\ProgramData\ssh\sshd_config`）

可以通过在 sshd_config 中设置以下内容来打开基于文件的日志记录选项`SyslogFacility  LOCAL0 ` ，使用此选项，日志将保存在` %programdata%\ssh\logs` 中

```apl
# Logging
#SyslogFacility AUTH
SyslogFacility LOCAL0 
```

更改 sshd_config 后重新启动 sshd 服务。

```powershell
Restart-Service sshd
```

修改`ipban.override.config` 文件

```xml
<configuration>
	<LogFilesToParse>        
    	<LogFiles>
    		<LogFile>
    			<!-- The file source (i.e. SSH, SMTP, etc.) -->
    			<Source>SSH</Source>

    			<PathAndMask>
                    <!-- 指定ssh日志文件 -->
        			C:\ProgramData\ssh\logs\*.log
    			</PathAndMask>

				<!-- 指定 platforms，（可用值：use Linux, MAC or Windows or Ubuntu, etc.） -->
        		<PlatformRegex>Windows</PlatformRegex>
            </LogFile>
        <LogFiles> 
	</LogFilesToParse>
            
    <appSettings>
        <!-- 失败次数 -->
        <add key="FailedLoginAttemptsBeforeBan" value="5"/>
        
        <!-- 封禁时长，DD:HH:MM:SS 格式。00:00:00:00 禁止 90 天（最长）-->
        <!-- 可以设置多个时间跨度,使用逗号分隔。必须是升序排列 -->
        <!-- 当一个被ban的ip地址BanTime到期后，再次出现failed login，就会将这个ip移动到下一个BanTime -->
        <add key="BanTime" value="00:00:60:00,01:00:00:00,30:00:00:00"/>
        
        <!-- 删除在此时间之后尚未被禁止的 failed logins 记录 -->
        <add key="ExpireTime" value="01:00:00:00"/>
        <!-- 白名单 -->
        <add key="WhitelistRegex" value="^(59\.56\.77\..*)|(156\.248\.72\..*)|(154\.82\.74\..*)$"/>	
    </appSettings>
<configuration>
```

```powershell
# 重启IPBan
Get-Service IPBan | Restart-Service
```



## 手动安装配置IPBan

如果你嫌powershell更新繁琐，你也可以下载官方windows安装包进行手动安装配置，解压zip包到`C:\Program Files\IPBan`， 手动配置对应服务，管理员打开cmd，输入指令：

```powershell
sc create IPBAN type=own start=auto binPath="C:\Program Files\IPBan\DigitalRuby.IPBan.exe DisplayName= IPBAN"
```

cmd命令查询需要执行的安全审计：

```powershell
auditpol /list /subcategory:* /r
```

列表中登录事件和登录/注销事件是需要记录的，实际上仅仅审计登录/注销事件即可，这个是用来记录登录ip，方便ipban获取登录ip进行对应ip封禁。
输入如下指令进行审计开启：

```powershell
auditpol /set /category:"{69979849-797A-11D9-BED3-505054503030}" /success:enable /failure:enable
auditpol /set /category:"{69979850-797A-11D9-BED3-505054503030}" /success:enable /failure:enable
```

计算机配置>Windows 设置>安全设置>本地策略>安全选项，修改以下项：

- 网络安全：限制NTLM：传入 NTLM流量 拒绝所有账户

手动配置 `C:\Software\IPBan\ipban.override.config` 定位appsettings项，修改次数3和ban的时间为60分钟，保存后重启服务，用一台机器测试ban是否生效。

## 卸载IPban

卸载的话，cmd输入：

```powershell
sc.exe stop IPBAN
sc.exe delete IPBAN
rmdir /s /q "C:/Program Files/IPBan"
```

#### Win 2012系统的配置

IPBan在windows 2012环境下需要一些基本配置才能正确生效，记录如下
官方的一键安装脚本需要power shell 5.1+版本，2012系统默认版本查询：
打开powershell输入$PSVersionTable, 发现PSVersion值为4.0

版本较低，需要手动升级：
https://docs.microsoft.com/en-us/powershell/scripting/windows-powershell/install/installing-windows-powershell?view=powershell-7.2

**温馨提示：**

安装WMF5.1需要系统 .net framework 4.5以上版本,核心功能需要4.5.2以上版本才能生效，而且IPBan运行一些功能也需要对应的4.5.2以上版本，如果你系统版本过低，访问微软下载安装对应版本：

https://docs.microsoft.com/en-us/dotnet/framework/install/guide-for-developers?redirectedfrom=MSDN

windows 2012 R2最高可安装4.6.2版本，但是貌似没有中文版本，在判断用户活跃状态时，IPBan调用对应dll库方法会出错，所以我们这里选择更新到4.5.2版本：

https://dotnet.microsoft.com/en-us/download/dotnet-framework/thank-you/net452-web-installer

安装好必备环境组件后，就可以直接使用powershell一键脚本进行IPBAN的安装了
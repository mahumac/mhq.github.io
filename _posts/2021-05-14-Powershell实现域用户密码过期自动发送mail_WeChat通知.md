---
title: Powershell实现域用户密码过期自动发送mail/WeChat通知
date: 2021-05-14 +0800
categories: [Windows, Powershell]
description: 
pin: false
tags: [Windows,Powershell] 
---
# Powershell实现域用户密码过期自动发送Mail / WeChat通知

```powershell
<#
#########################################################################
# For 1:检测AD密码过期时间，每周一、三、五发送企业微信通知
# For 4:自动修改用户Department信息
# For 5:自动修改账户mail为"%username%@domain.com"
# For 6:修改用户手机号码信息，同步Mobile、telephoneNumber
# For 7:设置用户UPN名为"$user@domain.com"
# For 8:检测没有登记Mobile的用户,每周一发送通知
# Version:1.1
# Time:2020.03.05
#########################################################################
#>

$ComputerName = Get-WmiObject -Class Win32_ComputerSystem | ForEach-Object{ $_.Name }

# 获取当前时间
$now = $Start = Get-Date
$now_day_a = $now.ToString("yyyyMMddhhmmss")
$now_day = $now.ToString("yyyy-MM-dd")

# 记录运行日志
$ResultLog = New-Item -Type File -Path "D:\log\" -Name SendWechatlog_$now_day_a.txt 
Start-Transcript -Path  "D:\log\WeChatresult.txt "  -ErrorAction:Continue -Force
# 删除 D:\Log目录下60天之前的日志
Get-ChildItem -Path D:\log\*.* -Recurse -ErrorAction:SilentlyContinue | `
    Where-Object -FilterScript{(((get-date) - ($_.CreationTime)).days -gt 30)} | `
    Remove-Item -Verbose 
# 删除 E:\LogFiles目录下30天之前的日志
Get-ChildItem -Path E:\LogFiles\*.* -Recurse -ErrorAction:SilentlyContinue | `
    Where-Object -FilterScript{(((get-date) - ($_.CreationTime)).days -gt 30)} | `
    Remove-Item -Verbose

function Send-WeChat 
{
    Param(
        [String]$corpid,
        [String]$corpsecret,
        [String]$Content,
        [String]$UserId,
        [String]$AgentId
    )
    $auth_string = "https://qyapi.weixin.qq.com/cgi-bin/gettoken?corpid=$corpid&corpsecret=$corpsecret"
    # 如果token缓存文件不存在 或者 缓存时间超过2小时（7200秒），则重新请求新token 并保存到本地
    if ( !(Test-Path .\token.txt) -or ( $(Get-Date) - (Get-Item .\token.txt).LastWriteTime ).TotalSeconds -ge 7200 )
    {
        $auth_values = Invoke-RestMethod $auth_string
        $token = $auth_values.access_token
        $token | Out-File .\token.txt -Force
    }else{
        $token = Get-Content .\token.txt
    }
    $body="{
        `"touser`":`"$UserId`",
        `"agentid`":`"$AgentId`",
        `"text`":{
            `"content`":`"$Content`"
        },
        `"msgtype`":`"text`"
    }"
    $Chinese=[System.Text.Encoding]::UTF8.GetBytes($body) # 中文字符编码
    Invoke-RestMethod `
        -Uri "https://qyapi.weixin.qq.com/cgi-bin/message/send?access_token=$token" `
        -ContentType "application/json" `
        -Method Post `
        -Body $Chinese
}
################################################
#                  调用示例                     #
################################################
$corpid="ww72c3199289a6eb5d" # 微信企业号的开发者ID
$corpsecret="_ga7oXckEHurG0ANQacc0JD7Gh_9kBRg742iD1o-WII" # 微信企业号的开发者密码
$AgentId="1000000" # 企业微信代理ID

# $Content="测试消息" #发送的文本信息
# $UserId="$user" #企业微信用户ID
# Send-WeChat -corpid $corpid -corpsecret $corpsecret -Content $Content -UserId $UserId -AgentId $AgentId


Import-Module Activedirectory
# 获取所有账户SAN名
$alladuser = Get-ADUser -Searchbase "DC=intra,DC=domain,DC=com" -filter * | ForEach-Object{ $_.SamAccountName }
# 统计用户数量
$userNum=0
$n1=0
$n2=0
$n3=0
$n4=0

$userlist = @()

foreach ($user in $alladuser)
{
    # 企业微信用户ID
    $UserId="$user" 

    # 获取账户显示名称（DisplayName）
    $chineseuserName=Get-ADUser $user  -Properties * | ForEach-Object{$_.Displayname}
    # 获取用户主体名UPN（UserPrincipalName）
    $UPN=Get-ADUser $user  -Properties * | ForEach-Object{$_.UserPrincipalName}
    # 获取用户电话号码(TelephoneNumber)
    $TelephoneNumber=Get-ADUser $user  -Properties * | ForEach-Object{$_.TelephoneNumber}
    # 获取用户移动电话(Mobile)
    $Mobile=Get-ADUser $user  -Properties * | ForEach-Object{$_.Mobile}
    # 获取用户备用邮箱(otherMailbox)
    $otherMailbox = Get-ADUser $user -Properties * | ForEach-Object{ $_.otherMailbox }
    # 账号创建时间
    $whenCreated = Get-ADUser $user -Properties * | ForEach-Object{ $_.whenCreated }
    $whenCreated_str = $whenCreated.ToString("yyyy-MM-dd HH:mm")

    ####################################################################################################
    #For 4:自动修改用户Department信息
    ####################################################################################################
    # 用户部门信息
    $Department = Get-ADUser $user -Properties * | ForEach-Object{ $_.$Department }
    # 根据帐号DistinguishedName信息（所在OU路径），预先生成Per_Department信息
    $user_DN = (Get-aduser $user -Properties *).DistinguishedName
    $split_DN = ( ( $user_DN -split "," ) -like "OU=*" ) -notlike "*四创集团*"  -replace( "OU=" , "")
    $Per_Department = $split_DN[-1] + " " + $split_DN[-2]
    $Department_split = ($Department -split " ")[-1]
    $DayOfWeek = "1"
    if( $Department_split -notlike "$($split_DN[-2])" -and (Get-Date).DayOfWeek -in $DayOfWeek )
    {
        Set-ADUser $user -Department $Per_Department
        Write-Output "修改账户 $chineseuserName($user) Department为：$Per_Department " | Out-File -FilePath $ResultLog -Append
    }

    # 获取帐户密码的最后一次更改时间(time-排除空值)，计算出账户密码的过期时间（day-过期日期）
    $pwdlastset = Get-ADUser $user -Properties * | ForEach-Object{ $_.PasswordLastSet }
    if ($null -ne $pwdlastset)
    {
        $pwdlastset_str = $pwdlastset.ToString("yyyy-MM-dd HH:mm")
        $pwdExpiredDate = ($pwdlastset).adddays(360)
        $pwdExpiredDate_str = $pwdExpiredDate.ToString("yyyy-MM-dd HH:mm")
    }
    else
    {
        $pwdExpiredDate = $now
        $pwdlastset_str = "从未修改过密码"
        $pwdExpiredDate_str = "-"
    }  

    # 判断账户是否设置了永不过期
    $neverexpire = Get-ADUser $user -Properties * | ForEach-Object{ $_.PasswordNeverExpires }
    # 判断账户是否已禁用状态和用户状态值
    $Enabled = Get-ADUser $user -Properties * | ForEach-Object{ $_.Enabled }
    # 距离密码过期的时间(天数)(若未过期，该值为正数；若已经过期，该值为负数)
    $expire_days = ($pwdExpiredDate - $now).Days

    ####################################################################################################
    #For 5:自动修改用户mail为"%username%@domain.com"
    ####################################################################################################
    $Mail = Get-ADUser $user -Properties * | ForEach-Object{ $_.Mail }
    if ($Mail -like '' -or $mail -like "*strongsoft*")
    {
        Set-ADUser $user -EmailAddress $user@domain.com
        Write-Output "设置 账户 $chineseuserName($user) 邮件地址为：$user@domain.com" | Out-File -FilePath $ResultLog -Append
    }

    ####################################################################################################
    #For 6:修改用户手机号码信息(部分属性参数（如：telephonenumber）无法使用set直接修改，需要用replace替换参数)
    ####################################################################################################
    if ($TelephoneNumber -like '' -and $Mobile -gt 0 )
    {
        Set-ADUser $user  -Replace @{telephonenumber= $Mobile}
        Write-Output "设置 账户 $chineseuserName($user) TelephoneNumber为：$Mobile" | Out-File -FilePath $ResultLog -Append
    }
    if ($Mobile -like '' -and $TelephoneNumber -gt 0 )
    {
        Set-ADUser $user -Mobile $TelephoneNumber
        Write-Output "设置 账户 $chineseuserName($user) Mobile为：$TelephoneNumber" | Out-File -FilePath $ResultLog -Append
    }

    ####################################################################################################
    #For 7:设置用户UPN名为"$user@domain.com"
    ####################################################################################################
    if ($UPN -like "*strongsoft*" -or $UPN -like "*@in.domain.com")
    {
        Set-ADUser $user -UserPrincipalName $user@domain.com
        Write-Output "修改用户 $chineseuserName($user) UPN名为：$user@domain.com" | Out-File -FilePath $ResultLog -Append
    }

    #####################################################################
    #判断过期时间小于15天(即将过期)、未禁用、且没有设置密码永不过期的账户
    ##For 1:检测AD密码过期时间并通知相应用户 (0 ≤ $expirt_days ≤ 15)
    #####################################################################
    if ($expire_days -ge 0 -and $expire_days -le 15 -and $neverexpire -like "false" -and $Enabled -like "True")
    {
        # 统计密码过期用户数量
        $userNum = $userNum + 1
         #########################
        #         消息正文         #
         #########################
        $Content =
        "亲爱的<$chineseuserName>同学 :
        您的邮箱<$user@domain.com>密码即将在 $expire_days 天后过期，$pwdExpiredDate_str 之后您的账号将被禁止登陆，请尽快更改密码。
        
        密码修改地址：http://password.domain.com.

        重置密码过程请遵循以下原则：
            ○ 密码长度最少 8 位；
            ○ 密码可使用最长时间 360天，过期需要更改密码；
            ○ 强制密码历史 1个（不能使用之前最近使用的 1 个密码）；
            ○ 密码符合复杂性需求（大写字母、小写字母、数字和符号四种中必须有三种、且不得包括用户名）
        "
        # 密码过期大于3天时，每周一、三、五，通知用户 (3 < $expirt_days ≤ 15)
        $DayOfWeek = "1,3,5"
        if( $expire_days -gt 3 -and (Get-Date).DayOfWeek -in $DayOfWeek ) 
        {
           Send-WeChat `
                -corpid $corpid `
                -corpsecret $corpsecret `
                -AgentId $AgentId `
                -UserId $UserId `
                -Content $Content
        }
        # 密码过期小于3天时，每天通知用户 (0 ≤ $expirt_days ≤ 3)
        if( $expire_days -le 3 ) 
        {
           Send-WeChat `
                -corpid $corpid `
                -corpsecret $corpsecret `
                -AgentId $AgentId `
                -UserId $UserId `
                -Content $Content
        }

        #####################################################################
        # 筛选$Mobile值为空、未被禁用、且密码未过期的帐号
        #For 8:检测没有登记Mobile的用户,每周一发送微信通知
        #####################################################################
        if ( (!$otherMailbox -or !$Mobile) -and $Enabled -and $expire_days -ge 0 )
        {
            # 统计未登记手机号码的用户数量
            $n4 = $n4+1
            ########################
            #         消息正文       #
            ########################
            $Content =
            "亲爱的 $chineseuserName 同学：
            你的邮箱 <$user@domain.com> 账号信息未登记完善，请登录 http://password.domain.com (建议使用电脑端登录)完善账号信息。
            以便在忘记密码或者账户被锁时可以自助重置密码或解锁账户。
                (a) 完善您的备用邮箱、手机号码
                (b) 配置您的个人认证安全问题及其答案
            "
            if ((Get-Date).DayOfWeek -eq 1)
            {
                # 延时一段时间
                Send-WeChat `
                    -corpid $corpid `
                    -corpsecret $corpsecret `
                    -AgentId $AgentId `
                    -UserId $UserId `
                    -Content $Content
            }
        }
         ########################
        #  查找账户的密码过期时间  #
         ########################
        $userName = Get-ADUser $user -Properties *
        $userobject = New-object psobject
        $userobject | Add-Member -membertype noteproperty -Name 用户名 -value $userName.displayname
        $userobject | Add-Member -membertype noteproperty -Name 邮箱 -Value $userName.mail
        $userobject | Add-Member -membertype noteproperty -Name 最近一次密码设置 -Value $pwdlastset_str
        $userobject | Add-Member -membertype noteproperty -Name 密码过期时间 -Value $pwdExpiredDate_str
        $userobject | Add-Member -membertype noteproperty -Name 创建日期 -Value $whenCreated_str
        $userobject | Add-Member -membertype noteproperty -Name 距离密码过期天数 -Value $expire_days       

        $userlist+= $userobject
    }
}

$User_sheet = $userlist | sort-object 最近一次登陆时间 | Format-Table | Out-String

$WeChatMessage =
    "$now_day 日`
    禁用$n3 个账户，总计禁用$n1+$n2 个账号。`
    密码已过期超过30天的 $n1 人`
    密码即将过期的 $userNum 人`
    标记为已离职的 $n2 人`
    未登记注册手机号码的 $n4 人 `
    " 
 #####################
# 发送微信通知给管理员 #
 #####################
 Send-WeChat `
    -corpid $corpid `
    -corpsecret $corpsecret `
    -AgentId $AgentId `
    -UserId "mhq" `
    -Content $WeChatMessage

Write-Output "$WeChatMessage" | Out-File -FilePath $ResultLog -Append
Write-Output "$user_sheet" | Out-File -FilePath $ResultLog -Append

$Stop = Get-Date
Write-Output "脚本总用时 $(($Stop - $Start).Minutes)分 $(($Stop - $Start).Seconds)秒" | Out-File -FilePath $ResultLog -Append

Stop-Transcript 
```


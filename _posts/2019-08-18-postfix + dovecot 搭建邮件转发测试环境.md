# postfix + dovecot 搭建邮件转发测试环境

修改时间 | 修改人 | 修改内容
--- | --- | ---
2019/08/18 | 于东亮 | 增加了更多的测试例子
2019/08/17 | 于东亮 | 加入 **配置说明** 这一节
2019/08/15 | 于东亮 | 多个 MTA 通过指定 MTA 转发
2019/08/14 | 于东亮 | 两个 MTA 通过指定 MTA 转发

[TOC]

## 测试要求
各个 MTA 间通过加密通道转发。

## 系统拓扑
``` dot
    digraph G {
        a -> r;
        r -> a;
        b -> r;
        r -> b;
        c -> r;
        r -> c;
        a[label="MTA1"]
        b[label="MTA2"]
        c[label="MTA3"]
        r[label="MTA relay"]
    }
```

* MTA1: 172.16.108.101，虚拟邮件域 vm1.vm。
* MTA2: 172.16.108.102，虚拟邮件域 vm2.vm。
* MTA3: 172.16.108.103，虚拟邮件域 vm3.vm。
* MTA relay: 172.16.108.100，虚拟邮件域 任意。

## 配置说明
* 整个环境未配置 DNS，均使用 IP 直接访问。
* 邮件域和主机的 ``` /etc/hostname ``` 没有关系。

### 使用 postfix 作为 MTA 进行邮件发送和转发
* postfix 使用 ``` /etc/postfix/main.cf ``` 中的 mydomain 来区分本地投递还是转发。
* MTA1/MTA2/MTA3 分别在三个不同的邮件域中。
* MTA1/MTA2/MTA3 都通过 MTA relay 转发邮件。

### 邮件账号为虚拟账号
* dovecot 使用 ``` /etc/dovecot/conf.d/auth-passwdfile.conf.ext ``` 来指定账号密码文件。
* postfix 使用 ``` /etc/postfix/main.cf ``` 中的 virtual_mailbox_maps 来指定账号文件。
* 需要认证的账号，postfix 和 dovecot 中必须配置一致。

### MTA1/MTA2/MTA3 上的 postfix 使用 dovecot 的 sasl 认证
* dovecot 在 ``` /etc/dovecot/conf.d/10-master.conf ``` 中打开对 postfix 的支持
``` vim
    service auth {
        unix_listener /var/spool/postfix/private/auth {
            mode = 0666
        }
    }
```

* postfix 使用 ``` /etc/postfix/main.cf ``` 指定使用 dovecot 的 sasl 认证
``` vim
    smtpd_sasl_type = dovecot
    smtpd_sasl_path = private/auth
```

* dovecot 和 postfix 中的 sasl path 必须配置一致。

### MTA relay 配置为无认证转发
postfix 在 ``` /etc/postfix/main.cf ``` 中指定可转发范围
``` vim
relay_domains = vm1.vm, vm2.vm, vm3.vm
```

### postfix 加密传输约束
在 ``` /etc/postfix/main.cf ``` 中
* 强制加密接收
``` vim
    smtpd_tls_security_level = encrypt
```

* 优先加密接收，协商失败转明文
``` vim
    smtpd_tls_security_level = may
```

* 强制加密中转
``` vim
    smtp_tls_security_level = encrypt
```

* 优先加密中转，协商失败转明文
``` vim
    smtp_tls_security_level = may
```

### postfix 发送认证约束
在 ``` /etc/postfix/main.cf ``` 中
* 非本地发送需要认证
``` vim
    smtpd_sender_restrictions = permit_auth_destination, permit_sasl_authenticated, reject
```

* 去除本地无认证特权
``` vim
    local_recipient_maps = ... (TBD)
```

### postfix 转发约束
在 ``` /etc/postfix/main.cf ``` 中
``` vim
smtpd_relay_restrictions = permit_auth_destination, reject
```

## 软件版本
* MTA1/MTA2/MTA3
    * CentOS 7.4.1708
    * postfix-2.10.1
    * dovecot-2.2.36
    * stunnel-4.56 (optional)

* MTA relay
    * CentOS 7.4.1708
    * postfix-3.4.6 （和我们自己保持相同的大版本）

## 测试账号
* 邮件域 vm1.vm (MTA1)

    账号 | 密码 | 账号 base64 | 密码 base64
    --- | --- | --- | ---
    alice@vm1.vm | alice | YWxpY2VAdm0xLnZt | YWxpY2U=
    test1@vm1.vm | test1 | dGVzdDFAdm0xLnZt | dGVzdDE=

* 邮件域 vm2.vm (MTA2)

    账号 | 密码 | 账号 base64 | 密码 base64
    --- | --- | --- | ---
    bob@vm2.vm | bob | Ym9iQHZtMi52bQ== | Ym9i
    test2@vm2.vm | test2 | dGVzdDJAdm0yLnZt | dGVzdDI=

* 邮件域 vm3.vm (MTA3)

    账号 | 密码 | 账号 base64 | 密码 base64
    --- | --- | --- | ---
    alpha@vm3.vm | alpha | YWxwaGFAdm0zLnZt | YWxwaGE=
    beta@vm3.vm | beta | YmV0YUB2bTMudm0= | YmV0YQ==

## MTA1

### 安装相关组件
``` shell
    yum install -y postfix dovecot stunnel telnet
```

* MTA 间通过 smtps/465 转发
    * postfix 2.x 不支持，需要使用 stunnel 转发。
    * postfix 3.x 支持。

* MTA 间通过端口 25/587 转发
    * postfix 2/3 都支持 STARTTLS。

* 如果不是指定**必须**使用 smtps/465 转发，stunnel 可以不装。
* telnet 用来做简单测试，可以不装。

### 放置 postfix/dovecot 使用的证书
把预先生成的 ca.crt, mta1.crt, mta1.key 拷贝到 ``` /etc/pki/mail/ ``` 目录下。
*如何生成服务器证书请自行查阅其他参考资料*

### 配置 dovecot
* 使用虚拟账号
    * 编辑 /etc/dovecot/vusers ``` vi /etc/dovecot/vusers ```，加入 vm1.vm 域中的账号
    ``` vim
        alice@vm1.vm:{Plain}alice::::::
        test1@vm1.vm:{Plain}test1::::::
    ```
    
    * 创建系统邮件用户，postfix/dovecot 使用此系统用户进行邮件读写
    ``` shell
        groupadd -g 5000 vmail
        useradd -g vmail -u 5000 vmail -d /var/mail/vhosts
    ```

* 编辑 /etc/dovecot/conf.d/10-auth.conf ``` vi /etc/dovecot/conf.d/10-auth.conf ```
``` vim
    # line 10，启用 PLAIN/LOGIN
    disable_plaintext_auth = no
    # line 100，增加 login 方式认证
    auth_mechanisms = plain login
    # line 122，注释掉，关闭系统账号认证
    #!include auth-system.conf.ext
    # line 125，打开注释，设置虚拟账号
    !include auth-passwdfile.conf.ext
    # line 128，打开注释，设置系统邮件用户
    !include auth-static.conf.ext
```

* 编辑 /etc/dovecot/conf.d/10-mail.conf ``` vi /etc/dovecot/conf.d/10-mail.conf ```
``` vim
    # line 30，设置虚拟账号邮件存储位置
    mail_location = maildir:/var/mail/vhosts/%d/%n
```

* 编辑 /etc/dovecot/conf.d/10-master.conf ``` vi /etc/dovecot/conf.d/10-master.conf ```
``` vim
    # line 96 - 98，打开注释，为 postfix 提供认证支持
        unix_listener /var/spool/postfix/private/auth {
            mode = 0666
        }
```

* 编辑 /etc/dovecot/conf.d/10-ssl.conf ``` vi /etc/dovecot/conf.d/10-ssl.conf ```
``` vim
    # line 14 - 15，指定 ssl 使用的证书
    ssl_cert = </etc/pki/mail/mta1.crt
    ssl_key = </etc/pki/mail/mta1.key
```

* 编辑 /etc/dovecot/conf.d/auth-passwdfile.conf.ext ``` vi /etc/dovecot/conf.d/auth-passwdfile.conf.ext ```
``` vim
    # line 8，指定虚拟账号密码文件
        args = /etc/dovecot/vusers
    # line 11 - 20，注释掉 userdb 这一节
    #userdb {
    # ...
    #}
```

* 编辑 /etc/dovecot/conf.d/auth-static.conf.ext ``` vi /etc/dovecot/conf.d/auth-static.conf.ext ```
``` vim
    # line 21 - 24，指定 dovecot 使用的系统用户，以及虚拟账号的邮件存储位置
    userdb {
        driver = static
        args = uid=vmail gid=vmail home=/var/mail/vhosts/%d/%n
    }
```

* 打开防火墙相关端口
``` shell
    firewall-cmd --add-port=110/tcp --permanent
    firewall-cmd --add-port=143/tcp --permanent
    firewall-cmd --add-port=993/tcp --permanent
    firewall-cmd --add-port=995/tcp --permanent
    firewall-cmd --reload
```

* 设置开机启动服务
``` shell
    systemctl enable dovecot
    systemctl restart dovecot
```

* 服务状态检查
``` shell
    systemctl status dovecot
    # 正常时存在 active (running) 字样
    ss -tnlp | grep dovecot
    # 正常时存在端口 110/995
```

* 虚拟账号检查
``` shell
    doveadm user alice@vm1.vm
    doveadm user test1@vm1.vm
```

### 配置 postfix
* 使用虚拟账号
    * 编辑 /etc/postfix/vusers ``` vi /etc/postfix/vusers ```，加入 vm1.vm 域中的账号
    ``` vim
        alice@vm1.vm vm1.vm/alice/
        test1@vm1.vm vm1.vm/test1/
    ```
    
    * 生成 /etc/postfix/vusers.db
    ``` shell
        postmap /etc/postfix/vusers
    ```

* 配置转发规则
*只有一个转发目的地的时候，也可以直接设置* **relayhost**
    * 编辑 /etc/postfix/ts ``` vi /etc/postfix/ts ```
        * smtps/465，使用 stunnel 转发
        ``` vim
            # 转发本地 stunnel
            vm2.vm  relay:127.0.0.1:25000
            vm3.vm  relay:127.0.0.1:25000
        ```

        * 端口 25/587，可以 STARTTLS 转加密
        ``` vim
            # 直接转发 MTA relay
            vm2.vm  relay:172.16.108.100
            vm3.vm  relay:172.16.108.100
            #vm2.vm  relay:172.16.108.100:587
            #vm3.vm  relay:172.16.108.100:587
        ```

    * 生成 /etc/postfix/ts.db
    ``` shell
        postmap /etc/postfix/ts
    ```

* 编辑 /etc/postfix/main.cf ``` vi /etc/postfix/main.cf ```
``` vim
    # line 76，设置主机名 [非必须]
    myhostname = mta1.vm1.vm
    # line 83，邮件域
    mydomain = vm1.vm
    # line 99
    myorigin = $mydomain
    # line 113, 116，监听所有地址
    inet_interfaces = all
    #inet_interfaces = localhost
    # -- 追加 --
    # 邮件最大尺寸 50M
    message_size_limit = 52428800
    # 邮箱 1G
    mailbox_size_limit = 1073741824
    # 虚拟用户
    virtual_mailbox_domains = $mydomain
    virtual_mailbox_base = /var/mail/vhosts
    virtual_mailbox_maps = hash:/etc/postfix/vusers
    virtual_mailbox_limit = $mailbox_size_limit
    virtual_uid_maps = static:5000
    virtual_gid_maps = static:5000  
    # dovecot sasl auth
    smtpd_sasl_auth_enable = yes
    smtpd_sasl_security_options = noanonymous
    smtpd_sasl_type = dovecot
    smtpd_sasl_path = private/auth
    smtpd_sasl_local_domain = $mydomain
    smtpd_sender_restrictions = permit_auth_destination, permit_sasl_authenticated, reject
    smtpd_recipient_restrictions = permit_auth_destination, permit_sasl_authenticated, reject
    broken_sasl_auth_clients = yes
    # 转发规则
    transport_maps = hash:/etc/postfix/ts
    # smtpd tls support
    smtpd_use_tls = yes
    smtpd_tls_security_level = may
    smtpd_tls_auth_only = yes
    smtpd_tls_loglevel = 1
    smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache
    smtpd_tls_CAfile = /etc/pki/mail/ca.crt  
    smtpd_tls_key_file = /etc/pki/mail/mta1.key
    smtpd_tls_cert_file = /etc/pki/mail/mta1.crt
    # smtp 转发 tls support
    smtp_use_tls = yes
    smtp_tls_security_level = encrypt
    smtp_tls_loglevel = 1
    smtp_tls_note_starttls_offer = yes  
    smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache
```

* 编辑 /etc/postfix/master.cf ``` vi /etc/postfix/master.cf ```
*可以使用 smtpd -v 记录更详细的日志*
``` vim
    # line 11，打开端口 25
    smtp      inet  n       -       n       -       -       smtpd
    # line 16，打开端口 587
    submission inet n       -       n       -       -       smtpd
    # line 26，打开端口 465
    smtps     inet  n       -       n       -       -       smtpd
    # line 28，stmp over tls support
        -o smtpd_tls_wrappermode=yes
```

* 打开防火墙相关端口
``` shell
    firewall-cmd --add-port=25/tcp --permanent
    firewall-cmd --add-port=465/tcp --permanent
    firewall-cmd --add-port=587/tcp --permanent
    firewall-cmd --reload
```

* 设置开机启动服务
``` shell
    systemctl enable postfix
    systemctl restart postfix
```

* 服务状态检查
``` shell
    systemctl status postfix
    # 正常时存在 active (running) 字样
    ss -tlnp | grep master
    # 正常时存在端口 25/465/587
```

### 本机收发邮件检查
* alice 发送给 test1
``` shell
    (
      echo "ehlo mta1"
      echo "auth login"
      echo "YWxpY2VAdm0xLnZt"
      echo "YWxpY2U="
      echo "mail from: alice"
      echo "rcpt to: test1"
      echo "data"
      echo "from: alice"
      echo "to: test1"
      echo "subject: alice to test1 on mta1"
      echo "alice say hello to test1"
      echo "."
      sleep 1
      echo "quit"
    ) | openssl s_client -crlf -connect 127.0.0.1:465
```

* 查看 /var/log/maillog 中相关日志。

* test1 查询邮件列表
``` shell
    (
      echo "user test1@vm1.vm"
      echo "pass test1"
      echo "list"
      sleep 1
      echo "quit"
    ) | openssl s_client -crlf -connect 127.0.0.1:995
```

* test1 查看第一封邮件
``` shell
    (
      echo "user test1@vm1.vm"
      echo "pass test1"
      echo "retr 1"
      sleep 1
      echo "quit"
    ) | openssl s_client -crlf -connect 127.0.0.1:995
```

### 配置 stunnel
* 编辑 /etc/stunnel/stunnel.conf ``` vi /etc/stunnel/stunnel.conf ```，转发到 MTA relay 的 smtps/465
``` vim
    [local-smtp-to-mta-smtps]
    accept = 25000
    client = yes
    connect = 172.16.108.100:465
```

* 启动 stunnel
``` shell
    stunnel
```

* 状态检查
``` shell
    ps -aux | grep stunnel
    # 正常时存在多个 stunnel 实例
```

## MTA2

### 安装相关组件
``` shell
    yum install -y postfix dovecot stunnel telnet
```

* MTA 间通过 smtps/465 转发
    * postfix 2.x 不支持，需要使用 stunnel 转发。
    * postfix 3.x 支持。

* MTA 间通过端口 25/587 转发
    * postfix 2/3 都支持 STARTTLS。

* 如果不是指定**必须**使用 smtps/465 转发，stunnel 可以不装。
* telnet 用来做简单测试，可以不装。

### 放置 postfix/dovecot 使用的证书
把预先生成的 ca.crt, mta2.crt, mta2.key 拷贝到 ``` /etc/pki/mail/ ``` 目录下。
*如何生成服务器证书请自行查阅其他参考资料*

### 配置 dovecot
* 使用虚拟账号
    * 编辑 /etc/dovecot/vusers ``` vi /etc/dovecot/vusers ```，加入 vm2.vm 域中的账号
    ``` vim
        bob@vm2.vm:{Plain}bob::::::
        test2@vm2.vm:{Plain}test2::::::
    ```
    
    * 创建系统邮件用户，postfix/dovecot 使用此系统用户进行邮件读写
    ``` shell
        groupadd -g 5000 vmail
        useradd -g vmail -u 5000 vmail -d /var/mail/vhosts
    ```

* 编辑 /etc/dovecot/conf.d/10-auth.conf ``` vi /etc/dovecot/conf.d/10-auth.conf ```
``` vim
    # line 10，启用 PLAIN/LOGIN
    disable_plaintext_auth = no
    # line 100，增加 login 方式认证
    auth_mechanisms = plain login
    # line 122，注释掉，关闭系统账号认证
    #!include auth-system.conf.ext
    # line 125，打开注释，设置虚拟账号
    !include auth-passwdfile.conf.ext
    # line 128，打开注释，设置系统邮件用户
    !include auth-static.conf.ext
```

* 编辑 /etc/dovecot/conf.d/10-mail.conf ``` vi /etc/dovecot/conf.d/10-mail.conf ```
``` vim
    # line 30，设置虚拟账号邮件存储位置
    mail_location = maildir:/var/mail/vhosts/%d/%n
```

* 编辑 /etc/dovecot/conf.d/10-master.conf ``` vi /etc/dovecot/conf.d/10-master.conf ```
``` vim
    # line 96 - 98，打开注释，为 postfix 提供认证支持
        unix_listener /var/spool/postfix/private/auth {
            mode = 0666
        }
```

* 编辑 /etc/dovecot/conf.d/10-ssl.conf ``` vi /etc/dovecot/conf.d/10-ssl.conf ```
``` vim
    # line 14 - 15，指定 ssl 使用的证书
    ssl_cert = </etc/pki/mail/mta2.crt
    ssl_key = </etc/pki/mail/mta2.key
```

* 编辑 /etc/dovecot/conf.d/auth-passwdfile.conf.ext ``` vi /etc/dovecot/conf.d/auth-passwdfile.conf.ext ```
``` vim
    # line 8，指定虚拟账号密码文件
        args = /etc/dovecot/vusers
    # line 11 - 20，注释掉 userdb 这一节
    #userdb {
    # ...
    #}
```

* 编辑 /etc/dovecot/conf.d/auth-static.conf.ext ``` vi /etc/dovecot/conf.d/auth-static.conf.ext ```
``` vim
    # line 21 - 24，指定 dovecot 使用的系统用户，以及虚拟账号的邮件存储位置
    userdb {
        driver = static
        args = uid=vmail gid=vmail home=/var/mail/vhosts/%d/%n
    }
```

* 打开防火墙相关端口
``` shell
    firewall-cmd --add-port=110/tcp --permanent
    firewall-cmd --add-port=143/tcp --permanent
    firewall-cmd --add-port=993/tcp --permanent
    firewall-cmd --add-port=995/tcp --permanent
    firewall-cmd --reload
```

* 设置开机启动服务
``` shell
    systemctl enable dovecot
    systemctl restart dovecot
```

* 服务状态检查
``` shell
    systemctl status dovecot
    # 正常时存在 active (running) 字样
    ss -tnlp | grep dovecot
    # 正常时存在端口 110/995
```

* 虚拟账号检查
``` shell
    doveadm user bob@vm2.vm
    doveadm user test2@vm2.vm
```

### 配置 postfix
* 使用虚拟账号
    * 编辑 /etc/postfix/vusers ``` vi /etc/postfix/vusers ```，加入 vm2.vm 域中的账号
    ``` vim
        bob@vm2.vm vm2.vm/bob/
        test2@vm2.vm vm2.vm/test2/
    ```
    
    * 生成 /etc/postfix/vusers.db
    ``` shell
        postmap /etc/postfix/vusers
    ```

* 配置转发规则
*只有一个转发目的地的时候，也可以直接设置* **relayhost**
    * 编辑 /etc/postfix/ts ``` vi /etc/postfix/ts ```
        * smtps/465，使用 stunnel 转发
        ``` vim
            # 转发本地 stunnel
            vm2.vm  relay:127.0.0.1:25000
            vm3.vm  relay:127.0.0.1:25000
        ```

        * 端口 25/587，可以 STARTTLS 转加密
        ``` vim
            # 直接转发 MTA relay
            vm1.vm  relay:172.16.108.100
            vm3.vm  relay:172.16.108.100
            #vm1.vm  relay:172.16.108.100:587
            #vm3.vm  relay:172.16.108.100:587
        ```

    * 生成 /etc/postfix/ts.db
    ``` shell
        postmap /etc/postfix/ts
    ```

* 编辑 /etc/postfix/main.cf ``` vi /etc/postfix/main.cf ```
``` vim
    # line 76，设置主机名 [非必须]
    myhostname = mta2.vm2.vm
    # line 83，邮件域
    mydomain = vm2.vm
    # line 99
    myorigin = $mydomain
    # line 113, 116，监听所有地址
    inet_interfaces = all
    #inet_interfaces = localhost
    # -- 追加 --
    # 邮件最大尺寸 50M
    message_size_limit = 52428800
    # 邮箱 1G
    mailbox_size_limit = 1073741824
    # 虚拟用户
    virtual_mailbox_domains = $mydomain
    virtual_mailbox_base = /var/mail/vhosts
    virtual_mailbox_maps = hash:/etc/postfix/vusers
    virtual_mailbox_limit = $mailbox_size_limit
    virtual_uid_maps = static:5000
    virtual_gid_maps = static:5000  
    # dovecot sasl auth
    smtpd_sasl_auth_enable = yes
    smtpd_sasl_security_options = noanonymous
    smtpd_sasl_type = dovecot
    smtpd_sasl_path = private/auth
    smtpd_sasl_local_domain = $mydomain
    smtpd_sender_restrictions = permit_auth_destination, permit_sasl_authenticated, reject
    smtpd_recipient_restrictions = permit_auth_destination, permit_sasl_authenticated, reject
    broken_sasl_auth_clients = yes
    # 转发规则
    transport_maps = hash:/etc/postfix/ts
    # smtpd tls support
    smtpd_use_tls = yes
    smtpd_tls_security_level = may
    smtpd_tls_auth_only = yes
    smtpd_tls_loglevel = 1
    smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache
    smtpd_tls_CAfile = /etc/pki/mail/ca.crt  
    smtpd_tls_key_file = /etc/pki/mail/mta2.key
    smtpd_tls_cert_file = /etc/pki/mail/mta2.crt
    # smtp 转发 tls support
    smtp_use_tls = yes
    smtp_tls_security_level = encrypt
    smtp_tls_loglevel = 1
    smtp_tls_note_starttls_offer = yes  
    smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache
```

* 编辑 /etc/postfix/master.cf ``` vi /etc/postfix/master.cf ```
*可以使用 smtpd -v 记录更详细的日志*
``` vim
    # line 11，打开端口 25
    smtp      inet  n       -       n       -       -       smtpd
    # line 16，打开端口 587
    submission inet n       -       n       -       -       smtpd
    # line 26，打开端口 465
    smtps     inet  n       -       n       -       -       smtpd
    # line 28，stmp over tls support
        -o smtpd_tls_wrappermode=yes
```

* 打开防火墙相关端口
``` shell
    firewall-cmd --add-port=25/tcp --permanent
    firewall-cmd --add-port=465/tcp --permanent
    firewall-cmd --add-port=587/tcp --permanent
    firewall-cmd --reload
```

* 设置开机启动服务
``` shell
    systemctl enable postfix
    systemctl restart postfix
```

* 服务状态检查
``` shell
    systemctl status postfix
    # 正常时存在 active (running) 字样
    ss -tlnp | grep master
    # 正常时存在端口 25/465/587
```

### 本机收发邮件检查
* bob 发送给 test2
``` shell
    (
      echo "ehlo mta2"
      echo "auth login"
      echo "Ym9iQHZtMi52bQ=="
      echo "Ym9i"
      echo "mail from: bob"
      echo "rcpt to: test2"
      echo "data"
      echo "from: bob"
      echo "to: test2"
      echo "subject: bob to test2 on mta2"
      echo "bob say hello to test2"
      echo "."
      sleep 1
      echo "quit"
    ) | openssl s_client -crlf -connect 127.0.0.1:465
```

* 查看 /var/log/maillog 中相关日志。

* test2 查询邮件列表
``` shell
    (
      echo "user test2@vm2.vm"
      echo "pass test2"
      echo "list"
      sleep 1
      echo "quit"
    ) | openssl s_client -crlf -connect 127.0.0.1:995
```

* test2 查看第一封邮件
``` shell
    (
      echo "user test2@vm2.vm"
      echo "pass test2"
      echo "retr 1"
      sleep 1
      echo "quit"
    ) | openssl s_client -crlf -connect 127.0.0.1:995
```

### 配置 stunnel
* 编辑 /etc/stunnel/stunnel.conf ``` vi /etc/stunnel/stunnel.conf ```，转发到 MTA relay 的 smtps/465
``` vim
    [local-smtp-to-mta-smtps]
    accept = 25000
    client = yes
    connect = 172.16.108.100:465
```

* 启动 stunnel
``` shell
    stunnel
```

* 状态检查
``` shell
    ps -aux | grep stunnel
    # 正常时存在多个 stunnel 实例
```

## MTA3

### 安装相关组件
``` shell
    yum install -y postfix dovecot stunnel telnet
```

* MTA 间通过 smtps/465 转发
    * postfix 2.x 不支持，需要使用 stunnel 转发。
    * postfix 3.x 支持。

* MTA 间通过端口 25/587 转发
    * postfix 2/3 都支持 STARTTLS。

* 如果不是指定**必须**使用 smtps/465 转发，stunnel 可以不装。
* telnet 用来做简单测试，可以不装。

### 放置 postfix/dovecot 使用的证书
把预先生成的 ca.crt, mta3.crt, mta3.key 拷贝到 ``` /etc/pki/mail/ ``` 目录下。
*如何生成服务器证书请自行查阅其他参考资料*

### 配置 dovecot
* 使用虚拟账号
    * 编辑 /etc/dovecot/vusers ``` vi /etc/dovecot/vusers ```，加入 vm3.vm 域中的账号
    ``` vim
        alpha@vm3.vm:{Plain}alpha::::::
        beta@vm3.vm:{Plain}beta::::::
    ```
    
    * 创建系统邮件用户，postfix/dovecot 使用此系统用户进行邮件读写
    ``` shell
        groupadd -g 5000 vmail
        useradd -g vmail -u 5000 vmail -d /var/mail/vhosts
    ```

* 编辑 /etc/dovecot/conf.d/10-auth.conf ``` vi /etc/dovecot/conf.d/10-auth.conf ```
``` vim
    # line 10，启用 PLAIN/LOGIN
    disable_plaintext_auth = no
    # line 100，增加 login 方式认证
    auth_mechanisms = plain login
    # line 122，注释掉，关闭系统账号认证
    #!include auth-system.conf.ext
    # line 125，打开注释，设置虚拟账号
    !include auth-passwdfile.conf.ext
    # line 128，打开注释，设置系统邮件用户
    !include auth-static.conf.ext
```

* 编辑 /etc/dovecot/conf.d/10-mail.conf ``` vi /etc/dovecot/conf.d/10-mail.conf ```
``` vim
    # line 30，设置虚拟账号邮件存储位置
    mail_location = maildir:/var/mail/vhosts/%d/%n
```

* 编辑 /etc/dovecot/conf.d/10-master.conf ``` vi /etc/dovecot/conf.d/10-master.conf ```
``` vim
    # line 96 - 98，打开注释，为 postfix 提供认证支持
        unix_listener /var/spool/postfix/private/auth {
            mode = 0666
        }
```

* 编辑 /etc/dovecot/conf.d/10-ssl.conf ``` vi /etc/dovecot/conf.d/10-ssl.conf ```
``` vim
    # line 14 - 15，指定 ssl 使用的证书
    ssl_cert = </etc/pki/mail/mta3.crt
    ssl_key = </etc/pki/mail/mta3.key
```

* 编辑 /etc/dovecot/conf.d/auth-passwdfile.conf.ext ``` vi /etc/dovecot/conf.d/auth-passwdfile.conf.ext ```
``` vim
    # line 8，指定虚拟账号密码文件
        args = /etc/dovecot/vusers
    # line 11 - 20，注释掉 userdb 这一节
    #userdb {
    # ...
    #}
```

* 编辑 /etc/dovecot/conf.d/auth-static.conf.ext ``` vi /etc/dovecot/conf.d/auth-static.conf.ext ```
``` vim
    # line 21 - 24，指定 dovecot 使用的系统用户，以及虚拟账号的邮件存储位置
    userdb {
        driver = static
        args = uid=vmail gid=vmail home=/var/mail/vhosts/%d/%n
    }
```

* 打开防火墙相关端口
``` shell
    firewall-cmd --add-port=110/tcp --permanent
    firewall-cmd --add-port=143/tcp --permanent
    firewall-cmd --add-port=993/tcp --permanent
    firewall-cmd --add-port=995/tcp --permanent
    firewall-cmd --reload
```

* 设置开机启动服务
``` shell
    systemctl enable dovecot
    systemctl restart dovecot
```

* 服务状态检查
``` shell
    systemctl status dovecot
    # 正常时存在 active (running) 字样
    ss -tnlp | grep dovecot
    # 正常时存在端口 110/995
```

* 虚拟账号检查
``` shell
    doveadm user alpha@vm3.vm
    doveadm user beta@vm3.vm
```

### 配置 postfix
* 使用虚拟账号
    * 编辑 /etc/postfix/vusers ``` vi /etc/postfix/vusers ```，加入 vm3.vm 域中的账号
    ``` vim
        alpha@vm3.vm vm3.vm/alpha/
        beta@vm3.vm vm3.vm/beta/
    ```
    
    * 生成 /etc/postfix/vusers.db
    ``` shell
        postmap /etc/postfix/vusers
    ```

* 配置转发规则
*只有一个转发目的地的时候，也可以直接设置* **relayhost**
    * 编辑 /etc/postfix/ts ``` vi /etc/postfix/ts ```
        * smtps/465，使用 stunnel 转发
        ``` vim
            # 转发本地 stunnel
            vm1.vm  relay:127.0.0.1:25000
            vm2.vm  relay:127.0.0.1:25000
        ```

        * 端口 25/587，可以 STARTTLS 转加密
        ``` vim
            # 直接转发 MTA relay
            vm1.vm  relay:172.16.108.100
            vm2.vm  relay:172.16.108.100
            #vm1.vm  relay:172.16.108.100:587
            #vm2.vm  relay:172.16.108.100:587
        ```

    * 生成 /etc/postfix/ts.db
    ``` shell
        postmap /etc/postfix/ts
    ```

* 编辑 /etc/postfix/main.cf ``` vi /etc/postfix/main.cf ```
``` vim
    # line 76，设置主机名 [非必须]
    myhostname = mta3.vm3.vm
    # line 83，邮件域
    mydomain = vm3.vm
    # line 99
    myorigin = $mydomain
    # line 113, 116，监听所有地址
    inet_interfaces = all
    #inet_interfaces = localhost
    # -- 追加 --
    # 邮件最大尺寸 50M
    message_size_limit = 52428800
    # 邮箱 1G
    mailbox_size_limit = 1073741824
    # 虚拟用户
    virtual_mailbox_domains = $mydomain
    virtual_mailbox_base = /var/mail/vhosts
    virtual_mailbox_maps = hash:/etc/postfix/vusers
    virtual_mailbox_limit = $mailbox_size_limit
    virtual_uid_maps = static:5000
    virtual_gid_maps = static:5000  
    # dovecot sasl auth
    smtpd_sasl_auth_enable = yes
    smtpd_sasl_security_options = noanonymous
    smtpd_sasl_type = dovecot
    smtpd_sasl_path = private/auth
    smtpd_sasl_local_domain = $mydomain
    smtpd_sender_restrictions = permit_auth_destination, permit_sasl_authenticated, reject
    smtpd_recipient_restrictions = permit_auth_destination, permit_sasl_authenticated, reject
    broken_sasl_auth_clients = yes
    # 转发规则
    transport_maps = hash:/etc/postfix/ts
    # smtpd tls support
    smtpd_use_tls = yes
    smtpd_tls_security_level = may
    smtpd_tls_auth_only = yes
    smtpd_tls_loglevel = 1
    smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache
    smtpd_tls_CAfile = /etc/pki/mail/ca.crt  
    smtpd_tls_key_file = /etc/pki/mail/mta3.key
    smtpd_tls_cert_file = /etc/pki/mail/mta3.crt
    # smtp 转发 tls support
    smtp_use_tls = yes
    smtp_tls_security_level = encrypt
    smtp_tls_loglevel = 1
    smtp_tls_note_starttls_offer = yes  
    smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache
```

* 编辑 /etc/postfix/master.cf ``` vi /etc/postfix/master.cf ```
*可以使用 smtpd -v 记录更详细的日志*
``` vim
    # line 11，打开端口 25
    smtp      inet  n       -       n       -       -       smtpd
    # line 16，打开端口 587
    submission inet n       -       n       -       -       smtpd
    # line 26，打开端口 465
    smtps     inet  n       -       n       -       -       smtpd
    # line 28，stmp over tls support
        -o smtpd_tls_wrappermode=yes
```

* 打开防火墙相关端口
``` shell
    firewall-cmd --add-port=25/tcp --permanent
    firewall-cmd --add-port=465/tcp --permanent
    firewall-cmd --add-port=587/tcp --permanent
    firewall-cmd --reload
```

* 设置开机启动服务
``` shell
    systemctl enable postfix
    systemctl restart postfix
```

* 服务状态检查
``` shell
    systemctl status postfix
    # 正常时存在 active (running) 字样
    ss -tlnp | grep master
    # 正常时存在端口 25/465/587
```

### 本机收发邮件检查
* alpha 发送给 beta
``` shell
    (
      echo "ehlo mta3"
      echo "auth login"
      echo "YWxpY2VAdm0xLnZt"
      echo "YWxpY2U="
      echo "mail from: alpha"
      echo "rcpt to: beta"
      echo "data"
      echo "from: alpha"
      echo "to: beta"
      echo "subject: alpha to beta on mta2"
      echo "alpha say hello to beta"
      echo "."
      sleep 1
      echo "quit"
    ) | openssl s_client -crlf -connect 127.0.0.1:465
```

* 查看 /var/log/maillog 中相关日志。

* beta 查询邮件列表
``` shell
    (
      echo "user beta@vm3.vm"
      echo "pass beta"
      echo "list"
      sleep 1
      echo "quit"
    ) | openssl s_client -crlf -connect 127.0.0.1:995
```

* beta 查看第一封邮件
``` shell
    (
      echo "user beta@vm3.vm"
      echo "pass beta"
      echo "retr 1"
      sleep 1
      echo "quit"
    ) | openssl s_client -crlf -connect 127.0.0.1:995
```

### 配置 stunnel
* 编辑 /etc/stunnel/stunnel.conf ``` vi /etc/stunnel/stunnel.conf ```，转发到 MTA relay 的 smtps/465
``` vim
    [local-smtp-to-mta-smtps]
    accept = 25000
    client = yes
    connect = 172.16.108.100:465
```

* 启动 stunnel
``` shell
    stunnel
```

* 状态检查
``` shell
    ps -aux | grep stunnel
    # 正常时存在多个 stunnel 实例
```

## MTA relay

### 安装 postfix3
* 添加 gf 源
``` shell
    rpm -ivh http://mirror.ghettoforge.org/distributions/gf/gf-release-latest.gf.el7.noarch.rpm
```

* 移除默认安装的 postfix
``` shell
    yum remove -y postfix
```

* 指定 gf 源安装 postfix3
``` shell
    yum --enablerepo=gf-plus install -y postfix3
```

### 放置 postfix使用的证书
把预先生成的 ca.crt, mta.crt, mta.key 拷贝到 ``` /etc/pki/mail/ ``` 目录下。
*如何生成服务器证书请自行查阅其他参考资料*

### 配置 postfix3
* 转发设置
*使用转发配置文件，不同的邮件域转发到对应的 MTA。*
    * 编辑 /etc/postfix/ts ``` vi /etc/postfix/ts ```
        * smtps/465
        ``` vim
            vm1.vm  relay-smtps:172.16.108.101
            vm2.vm  relay-stmps:172.16.108.102
            vm3.vm  relay-stmps:172.16.108.103
        ```
 
        * 端口 25/587，STARTTLS
        ``` vim
            vm1.vm  relay:172.16.108.101
            vm2.vm  relay:172.16.108.102
            vm3.vm  relay:172.16.108.103
        ```
    
    * 生成 /etc/postfix/ts.db
    ``` shell
        postmap /etc/postfix/ts
    ```

* 编辑 /etc/postfix/main.cf ``` vi /etc/postfix/main.cf ```
``` vim
    # line 95，设置主机名 [非必须]
    myhostname = mta.vm.vm
    # line 102，设置本机邮件域 [非必须]
    mydomain = vm.vm
    # line 132, 135，监听所有地址
    inet_interfaces = all
    #inet_interfaces = localhost
    # line 315，设置转发范围
    relay_domains = vm1.vm, vm2.vm, vm3.vm
    # -- 追加 --
    # 邮件 50M
    message_size_limit = 52428800
    # 邮箱 1G
    mailbox_size_limit = 1073741824
    # 转发映射
    transport_maps = hash:/etc/postfix/ts
    # smtpd tls support
    smtpd_use_tls = yes
    smtpd_tls_security_level = may
    smtpd_tls_loglevel = 1
    smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache
    smtpd_tls_CAfile = /etc/pki/mail/ca.crt  
    smtpd_tls_key_file = /etc/pki/mail/mta.key
    smtpd_tls_cert_file = /etc/pki/mail/mta.crt
    # smtp 转发 tls support
    smtp_use_tls = yes
    smtp_tls_security_level = encrypt
    smtp_tls_loglevel = 1
    smtp_tls_note_starttls_offer = yes  
    smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache
```

* 编辑 /etc/postfix/master.cf ``` vi /etc/postfix/master.cf ```
*可以使用 smtpd -v 记录更详细的日志*
``` vim
    # line 12，打开端口 25
    smtp      inet  n       -       n       -       -       smtpd
    # line 17，打开端口 587
    submission inet n       -       n       -       -       smtpd
    # line 29，打开端口 465
    smtps     inet  n       -       n       -       -       smtpd
    # line 31 [非必须]
        -o smtpd_tls_wrappermode=yes
```

* 打开防火墙相关端口
``` shell
    firewall-cmd --add-port=25/tcp --permanent
    firewall-cmd --add-port=465/tcp --permanent
    firewall-cmd --add-port=587/tcp --permanent
    firewall-cmd --reload
```

* 设置开机启动服务
``` shell
    systemctl enable postfix
    systemctl restart postfix
```

* 服务状态检查
``` shell
    systemctl status postfix
    # active (running)
    ss -tlnp | grep master
    # 端口 25/465
```

## 跨域收发邮件检查

### vm1.vm => vm2.vm
* 现在，alice 终于可以通过 MTA1 给 bob 发邮件了
``` shell
    (
      echo "ehlo mta1"
      echo "auth login"
      echo "YWxpY2VAdm0xLnZt"
      echo "YWxpY2U="
      echo "mail from: alice"
      echo "rcpt to: bob@vm2.vm"
      echo "data"
      echo "from: alice"
      echo "to: bob@vm2.vm"
      echo "subject: alice to bob"
      echo "alice say hello to bob"
      echo "."
      sleep 1
      echo "quit"
    ) | openssl s_client -crlf -connect 172.16.108.101:465
```

* 查看 172.16.108.101 上 /var/log/maillog 中相关日志。

* 查看 172.16.108.100 上 /var/log/maillog 中相关日志。

* 查看 172.16.108.102 上 /var/log/maillog 中相关日志。

* bob 在 MTA2 上查询邮件列表
``` shell
    (
      echo "user bob@vm2.vm"
      echo "pass bob"
      echo "list"
      sleep 1
      echo "quit"
    ) | openssl s_client -crlf -connect 172.16.108.102:995
```

* bob 在 MTA2 上查看第一封邮件
``` shell
    (
      echo "user bob@vm2.vm"
      echo "pass bob"
      echo "retr 1"
      sleep 1
      echo "quit"
    ) | openssl s_client -crlf -connect 172.16.108.102:995
```

### vm2.vm => vm3.vm
* 尝试 bob 通过 MTA2 发送给 beta
``` shell
    (
      echo "ehlo mta2"
      echo "auth login"
      echo "Ym9iQHZtMi52bQ=="
      echo "Ym9i"
      echo "mail from: bob"
      echo "rcpt to: beta@vm3.vm"
      echo "data"
      echo "from: bob"
      echo "to: beta@vm3.vm"
      echo "subject: bob to beta"
      echo "bob say hello to beta"
      echo "."
      sleep 1
      echo "quit"
    ) | openssl s_client -crlf -connect 172.16.108.102:465
```

* 查看 172.16.108.102 上 /var/log/maillog 中相关日志。

* 查看 172.16.108.100 上 /var/log/maillog 中相关日志。

* 查看 172.16.108.103 上 /var/log/maillog 中相关日志。

* beta 在 MTA3 上查询邮件列表
``` shell
    (
      echo "user beta@vm3.vm"
      echo "pass beta"
      echo "list"
      sleep 1
      echo "quit"
    ) | openssl s_client -crlf -connect 172.16.108.103:995
```

* alice 在 MTA1 上查看第二封邮件（如果之前按文档进行过 MTA2 的本机测试的话）
``` shell
    (
      echo "user beta@vm3.vm"
      echo "pass beta"
      echo "retr 2"
      sleep 1
      echo "quit"
    ) | openssl s_client -crlf -connect 172.16.108.103:995
```

### vm3.vm => vm1.vm
* 尝试 alpha 通过 MTA3 发送给 alice
``` shell
    (
      echo "ehlo mta3"
      echo "auth login"
      echo "YWxwaGFAdm0zLnZt"
      echo "YWxwaGE="
      echo "mail from: alpha"
      echo "rcpt to: alice@vm1.vm"
      echo "data"
      echo "from: alpha"
      echo "to: alice@vm1.vm"
      echo "subject: alpha to alice"
      echo "alpha say hello to alice"
      echo "."
      sleep 1
      echo "quit"
    ) | openssl s_client -crlf -connect 172.16.108.103:465
```

* 查看 172.16.108.103 上 /var/log/maillog 中相关日志。

* 查看 172.16.108.100 上 /var/log/maillog 中相关日志。

* 查看 172.16.108.101 上 /var/log/maillog 中相关日志。

* alice 在 MTA1 上查询邮件列表
``` shell
    (
      echo "user alice@vm1.vm"
      echo "pass alice"
      echo "list"
      sleep 1
      echo "quit"
    ) | openssl s_client -crlf -connect 172.16.108.101:995
```

* alice 在 MTA1 上查看第一封邮件
``` shell
    (
      echo "user alice@vm1.vm"
      echo "pass alice"
      echo "retr 1"
      sleep 1
      echo "quit"
    ) | openssl s_client -crlf -connect 172.16.108.101:995
```

## 异常操作检查
* 尝试无认证跨域发送邮件 [alice => bob NOT on MTA1]
``` shell
    (
      echo "ehlo mta1"
      echo "mail from: alice"
      echo "rcpt to: bob@vm2.vm"
      echo "data"
      echo "from: alice"
      echo "to: bob@vm2.vm"
      echo "subject: alice to bob without auth"
      echo "alice say hello to bob without auth"
      echo "."
      sleep 1
      echo "quit"
    ) | openssl s_client -crlf -connect 172.16.108.101:465
```

* 尝试无认证本机发送邮件 [alice => test1 on MTA1]
``` shell
    (
      echo "ehlo mta1"
      echo "mail from: alice"
      echo "rcpt to: test1"
      echo "data"
      echo "from: alice"
      echo "to: test1"
      echo "subject: alice to test1 without auth"
      echo "alice say hello to test1 without auth"
      echo "."
      sleep 1
      echo "quit"
    ) | openssl s_client -crlf -connect 127.0.0.1:465
```


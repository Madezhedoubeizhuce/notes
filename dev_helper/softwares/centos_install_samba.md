# 关闭SELinux和防火墙

1、临时关闭（不用重启机器）: 

```bash
setenforce 0                       #设置SELinux 成为permissive模式  （关闭SELinux）
setenforce 1                       #设置SELinux 成为enforcing模式     （开启SELinux）1
```

2、修改配置文件需要重启机器：

```bash
vi /etc/selinux/config
```

将SELINUX=enforcing 改为SELINUX=disabled(需重启机器)

3、可自己做策略开放相应端口，这里直接关闭:

```bash
systemctl status firewalld.service       #查看防火墙状态
systemctl stop firewalld.service         #关闭防火墙
systemctl disable firewalld.service		 #禁用防火墙
```

# 安装Samba服务

1、直接yum安装

```bash
yum install samba
```

2、启动并查看Samba

```bash
systemctl start smb
systemctl status smb
```

# 配置Samba服务

1、备份默认配置文件

```bash
cp /etc/samba/smb.conf /etc/sambasmb.conf.bak
```

2、使用`smb.conf.example`作为模板，即将`smb.conf.example`作为配置文件

```bash
cp /etc/samba/smb.conf.example /etc/sambasmb.conf
```

3、在`smb.conf`后添加配置

```bash
[alpha]
   comment = Alpha's stuff
   path = /home/alpha
   valid users = alpha
   public = no
   writable = yes
   printable = no
   create mask = 0765
```

# 添加samba用户

假设linux中已存在alpha用户，直接用如下命令添加对应的samba用户

```bash
smbpasswd -a alpha
```

# 重启samba服务

```bash
systemcl restart smb
```


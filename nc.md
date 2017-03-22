# nc 的使用

### 简单介绍
- netcat是网络工具中的瑞士军刀，它能通过TCP和UDP在网络中读写数据。通过与其他工具结合和重定向，你可以在脚本中以多种方式使用它。使用netcat命令所能完成的事情令人惊讶。

## 功能演示

### １端口扫描
> nc -zvn 122.112.13.132 80-110
> z 参数告诉netcat使用0 IO,连接成功后立即关闭连接， 不进行数据交换
> v 详细输出
> n 不用dns 解析
> Connection to 127.0.0.1 3306 port [tcp/mysql] succeeded!
L
5.6.30-1,UX<_L>3%��\"2v{(O9k2V9mysql_native_password

### 2　Chat Server(显示，回显)
> server:	nc -l 22222
clinet:	 nc 192.168.3.222 22222 
连接成功后，任何一方输入，２端显示都是一样的

### 3 文件传输
- server
nc -l 1567 < file.txt
- client
nc -n 192.168.3.222 1567 > file.txt
- 反向传输
nc -l 1567 > file.txt
nc -n 192.168.3.222 1567 < file.txt

### 4 目录传输
- server
tar -cvf – dir_name| bzip2 -z | nc -l 1567 
- client
nc -n 192.168.3.222 1567 | bzip2 -d |tar -xvf -

### 5 加密通信数据
nc localhost 1567 | mcrypt –flush –bare -F -q -d -m ecb > file.txt
mcrypt –flush –bare -F -q -m ecb < file.txt | nc -l 1567

### 6 克隆设备
mcrypt –flush –bare -F -q -m ecb < file.txt | nc -l 1567
nc -n 192.168.1.63 1567 | dd of=/dev/sda

### 7 打开shell 
- 把本地的shell 放入nc 监听的端口
nc -l 1567 -e /bin/bash -i
- 连接shell
nc 192.168.3.222 1567

- nc 的版本可能不一样
nc.traditional -lvvp 1567 -e /bin/bash 
nc 192.168.3.222 1567
- 另一种方法
mkfifo /tmp/tmp_fifo
cat /tmp/tmp_fifo | /bin/bash -i 2>&1 | nc -l 1567 > /tmp/tmp_fifo
nc 192.168.3.222 1567　有提示符,sh不能使用sudo ，为什么还暂时不知道 bash 可以




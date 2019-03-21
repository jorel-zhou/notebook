---
title: "CLI Cheetsheet"
permalink: /memo/memo-cli-cheetsheet/
excerpt: "CLI Cheetsheet"
last_modified_at: 2018-11-25T22:21:33-05:00
redirect_from:
  - /theme-setup/
toc: true
---

# CLI Cheat Sheet

# Index
1. [Certificate CLIs](#certificate-clis)
2. [Linux Server Network](#linux-server-network)
3. [Linux Server Performance Tunning](#linux-server-performance-tunning)
3. [Database Management](#database-management)
    * [PostgreSQL](#postgresql)

# Certificate CLIs
### basic keytool operation
```bash
keytool -list -v -keystore sample.keystore -storepass sample.pass
keytool -export -alias sample.alias -keystore sample.keystore -file sample.crt -storepass sample.pass
keytool -printcert -file sample.crt
keytool -import -alias sample.alias -file sample.crt -keystore sample.keystore -storepass
keytool -storepasswd -keystore sample.keystore -storepass sample.pass -new new.pass
keytool -delete -alias sample.alias -keystore sample.keystore -storepass sample.pass
openssl s_client -host google.com -port 443 -prexit -showcerts
```
### make keystore with pem certificate
```
cat myhost.pem intermediate.pem root.pem > import.pem
openssl pkcs12 -export -in import.pem -inkey myhost.key.pem -name shared > server.p12
keytool -importkeystore -srckeystore server.p12 -destkeystore store.keys -srcstoretype pkcs12 -alias shared
```

### change keypassword
```
keytool -keypasswd -alias alias.sample -keypass sample.pass -new sample.pass  -keystore .keystore -storepass sample.pass

# Verify key/cert pair matching, follow value should be same (first for the certificate file;second for the key file)
openssl x509 -noout -modulus -in <certificate>.pem | openssl md5
openssl rsa -noout -modulus -in <encrypted>.key | openssl md5
```

### check/get server side certificate
```
echo | openssl s_client -servername server -connect server:port 2>/dev/null | openssl x509 -text
echo | openssl s_client -connect server:port 2>&1 | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > cert.pem

openssl x509 -in sample.crt -text -noout
openssl s_client -showcerts -verify 5 -connect www.domain.com:443 < /dev/null

/etc/pki/ca-trust/source/anchors/ update-ca-trust enable; update-ca-trust extract awk -v cmd='openssl x509 -noout -subject' '/BEGIN/{close(cmd)};{print | cmd}' < /etc/ssl/certs/ca-bundle.crt
```

### convert certificate
```
openssl x509 -in cert.pem -inform PEM -out cert.der -outform DER
```


# Linux Server Network
### check tcp connections 
```
netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
netstat -nat|grep ESTABLISHED|wc -l
netstat -nat|grep -i "9444"|wc -l
lsof -i|grep propel|grep -v tokenidm|grep atc-cr-mongodb1.ecs-core.ssn.hp.com:postgres|wc -l
ps aux|grep -v grep|awk '/java/{sum+=$6;n++};END{print sum/n}'

CLOSED：无连接是活动的或正在进行 
LISTEN：服务器在等待进入呼叫 
SYN_RECV：一个连接请求已经到达，等待确认 
SYN_SENT：应用已经开始，打开一个连接 
ESTABLISHED：正常数据传输状态 
FIN_WAIT1：应用说它已经完成 
FIN_WAIT2：另一边已同意释放 
ITMED_WAIT：等待所有分组死掉 
CLOSING：两边同时尝试关闭 
TIME_WAIT：另一边已初始化一个释放 
LAST_ACK：等待所有分组死掉
```

# Linux Server Performance Tunning
### check linux server performance
```
cat /proc/cpuinfo |grep "physical id"|sort |uniq|wc -l
cat /proc/cpuinfo |grep "processor"|wc -l
cat /proc/cpuinfo |grep "cores"|uniq
cat /proc/cpuinfo |grep MHz|uniq
grep 'model name' /proc/cpuinfo | wc -l
vmstat 1
free -h
top
iostat -x 1 2
watch -d pstree -u propel
sar -b 5 5        // IO传送速率
sar -B 5 5        // 页交换速率
sar -c 5 5        // 进程创建的速率
sar -d 5 5        // 块设备的活跃信息
sar -n DEV 5 5    // 网路设备的状态信息
sar -n SOCK 5 5   // SOCK的使用情况
sar -n ALL 5 5    // 所有的网络状态信息
sar -P ALL 5 5    // 每颗CPU的使用状态信息和IOWAIT统计状态 
sar -q 5 5        // 队列的长度（等待运行的进程数）和负载的状态
sar -r 5 5       // 内存和swap空间使用情况
sar -R 5 5       // 内存的统计信息（内存页的分配和释放、系统每秒作为BUFFER使用内存页、每秒被cache到的内存页）
sar -u 5 5       // CPU的使用情况和IOWAIT信息（同默认监控）
sar -v 5 5       // inode, file and other kernel tablesd的状态信息
sar -w 5 5       // 每秒上下文交换的数目
sar -W 5 5       // SWAP交换的统计信息(监控状态同iostat 的si so)
sar -x 2906 5 5  // 显示指定进程(2906)的统计信息，信息包括：进程造成的错误、用户级和系统级用户CPU的占用情况、运行在哪颗CPU上
sar -y 5 5       // TTY设备的活动状态
sysctl  -a | grep inode
sysctl -p
```

# Database Management

# PostgreSQL
### check postgresql DB connections
```
pg_ctl status
ps -ef |grep postgres |wc -l
select count(1) from pg_stat_activity;
show max_connections;
SELECT count(*) FROM pg_stat_activity WHERE NOT pid=pg_backend_pid();
select state from pg_stat_activity where datname = 'sx';
```
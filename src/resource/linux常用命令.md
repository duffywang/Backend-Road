
```$xslt
ls -a //显示隐藏的文件
ls -l //显示文件详细信息，包括权限，软链接 
ls -l opt //查看opt目录的链接
ls -lh //文件适合的大小
```


```
cd //回到主目录
cd ..//回到父目录
cd - //回到你操作的过下一个目录中
cd /data/applogs //日志目录
```


```$xslt
ifconfig 
ifconfig | grep inet   //本机ip地址
```

```$xslt
chmod 777 text.txt
chmod u=rwx,g=rw,o=r text.txt  //u 表示拥有者，g表示群组 ，o代表其他 ，x表示执行或切换的权限
```

```$xslt
ps //静态显示进程
ps aux | sort -rnk 4: //按内存资源的使用量对进程进行排序
ps aux | sort -nk 3: //按CPU资源的使用量对进程进行排序
ps -ef | grep java  //进程中包含java的应用，
top //动态显示进程情况
```

CPU耗时高分析

```$xslt
ps -ef |grep java //找出对应java进程pid
top -H -p <pid> //找出进程中最消耗CPU的线程，将找出的线程id转换为16进制
jstack <pid> //获取java的线程堆栈
```

查看某个端口号被某个线程占用 （list open files）

```$xslt
lsof -i 
lsof -i :1099 //查看占用1099端口的进程
lsof -w -n -i tcp:8080 //查看8080端口
kill -9 pid //强制杀死线程
```

find使用

```$xslt
find . -name "*.c"
find /var/log -type f // var/log目录中一般文件类型
find . -ctime -20 //查找当前目录中20天内更新过的文件列出
find . -perm 644 //权限
```


如何查看日志

```$xslt
cat -n apiSys.log | grep "exception" //-n显示行数 

tail -500 /data/applogs/tomcat/catalina.out 启动报错日志

grep "WARN" apisys.log 

grep -n "NullPointerException" error.log 行数

grep -m 1 "NullPointerException" error.log 返回第一次匹配的

grep "WARN" apisys.log -nR --color  带颜色返回  

grep -i  忽略字符大小写的差别

grep -C 20 "SEVERE" localhost.2020-02-26.log -nR --color// 前后20行

head -n 100 apisys.log //前100行

tail -n 100 apisys.log //后100行

查看tomcat最新日志

./catalina.sh run
```

删除日志

```$xslt
pssh dictionary 'sudo rm -f xx.log && echo "ok"'

```
磁盘

```$xslt
df -h 查看/opt使用情况
du -s /opt/logs/*  |  sort -rn 查看当前目录占用最大的文件
```

```$xslt
curl -X HEAD -I 10.73.208.202
curl -v IP
```


网络性能

```
netstat -tn //连接状态、连接数、发送/接收队列
netstat -s | grep “restransimt” //网络重传统计
```


jvm

```$xslt
 jstat [-命令选项] [vmid] [间隔时间/毫秒] [查询次数] 
 jstat -gcutil 2206 10000 10  //  以查看堆内存各部分的使用量，以及加载类的数量
 
 jmap -heap 2206 //堆内存信息,包括jvm值的设置、年轻代、年老代大小
 
 jstack [ option ] pid  //java进程详细堆栈信息，如死锁，死循环等
```



# rsyslog 核心概念

**rsyslog** 日志是使用 **rsyslogd** 守护进程进行管理的，该进程是之前版本的系统中 **syslogd** 的升级版，对原有的日志系统进行了功能的扩展，提供了诸如**过滤器**、**日志加密保护**、**各种配置选项**、**输入输出模块**，支持通过 **TCP** 或者 **UDP** 协议进行传输等。

**rsyslog** 的配置文件为 **/etc/rsyslog.conf** , 大多数日志文件都位于 **/var/log/** 目录中。

## 格式

```
FACILITY.PRIORITY	action
```
- FACILITY	设施、服务名称
- PRIORITY	优先级、级别
- FACILITY.PRIORITY = 基于优先级的过滤器



## 服务名称（FACILITY）

|  序号  |  类别  |  说明  |
| --- | --- | --- |
| 0	   |   `kern`(`kernel`)	   |   内核信息  | 
| 1   |   	`user`   |   	用户相关  | 
| 2   |   	`mail`   |   	邮件收发信息  | 
| 3   |   	`daemon`   |   	守护进程、系统服务产生的信息  | 
| 4   |   	`auth`   |   	认证、授权，例如 login, ssh, su 等需要账户密码的东东  | 
| 5   |   	`syslog`   |   	syslog 自己的日志  | 
| 6   |   	`lpr`   |   	打印机相关  | 
| 7   |   	`news`   |   	新闻先关  | 
| 8   |   	`uucp`   |   	全名为 Unix to Unix Copy Protocol，早期用与 unix 系统间信息交换  | 
| 9   |   	`cron`   |   	任务相关  | 
| 10   |   	`authpriv`   |   	与 auth 类似，但记录较多帐号私人的信息，包括pam模块的运作等！  | 
| 11   |   	`ftp`   |   	ftp 服务相关  | 
| 16~23   |   	`local0 ~ local7`	   |   预留给用户自己定义使用的  | 
|    |   	`*`	   |   所有的 FACILITY  | 

## 级别

|  数值等级  |  等级名称  |  说明  |
| --- | --- | --- |
|    |  none  |  不需要记录日志  |
|    |  *  |  所有日志级别  |
|  7  |  debug  |  调试日志  |
||||
|  6  |  info  |  基本说明信息  |
|  5  |  notice  |  相对比info 稍微重要的信息  |
|  4  |  warning(warn)  |  可能有问题，但是还不至于影响到某个 daemon 的运作；info/notice/warn 这三个级别都是在告知一些还不至于造成系统运行的问题  |
||||
|  3  |  err(error)  |  错误信息，告知该错误会在成服务无法启动  |
|  2  |  crit  |  比error更严重的错误信息  |
|  1  |  alert  |  已经很验证的问题了，比crit更严重  |
||||
|  0  |  emerg  |  系统几乎要挂了，常用于硬件问题  |

## 特殊符号

- `.` ：比后面还要严重的等级要记录下来；例如 `mail.info` 代表只要 mail 的日志，并且记录等级 >=info 级别的日志
- `.=`：只记录当前等级的日志
- `.!`：低于该等级的日志会被记录
- `.!=`：非该级别的日志

## action 日志记录的位置

- `绝对路径`： 本地，如 /var/log/xxx
- `|` : 通过管道发给给其他命令
- `终端`： 如 /dev/console
- `@HOST`： 远程主机
- `*` ： 登陆到系统上所有用户，一般 emerg 级别的日志是这样定义的

## 示例

``` bash
# mail 相关，级别 >= info 的日志记录到 /var/log/mail.log 文件中
mail.info	/var/log/mail.log	

# 权限相关，级别为 info 的日志记录到 10.10.3.33 机器上，前提是 10.10.3.33 可以接收
auth.=info	@10.10.3.33

# 记录user 相关的日志，但是不记录 error 级别的
user.!=error	/var/log/xxx

# 与 user.error 相反
user.!error	/var/log/xxx

# 所有设备 >= info 级别的日志
*.info		/var/log/xxx

# mail 相关所有级别的日志
mail.*	/var/log/xxx

# 所有设备 所有级别的日志
*.*	/var/log/xxx

# 多个日志来源用 ; 分割。 也可以写成 user,mail.info	/var/log/xxx
user.info;mail.info	/var/log/xxx

# 选择除了 info 和 debug 优先级的 cron 日志
cron.!info,!debug	/var/log/xxx

# 写日志的时候不直接写入磁盘，而是先写入buffer，再批量写入磁盘，系统异常关闭的时候可能会丢日志
mail.info	 -/var/log/mail.log

```


## Read More
- 鸟哥 - [第十八章、認識與分析登錄檔](http://linux.vbird.org/linux_basic/0570syslog.php)
- [Linux环境下使用rsyslog管理日志](https://segmentfault.com/a/1190000003509909)
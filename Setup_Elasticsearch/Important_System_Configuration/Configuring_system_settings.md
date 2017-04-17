# 系统设置

在哪里配置系统设置取决于你使用哪个包安装Elasticsearch，以及您正在使用的操作系统。

当你使用`.zip`或`.tar.gz`的安装包是，系统设置可以按如下方式配置：

* 通过[ulimit](#ulimit)临时限制
* 通过[/etc/security/limits.conf](#limits.conf)永久限制

当你使用RPM或Debian安装包时，更多的系统设置可以通过(系统配置文件](#sysconfig)。但是，使用systemd的系统设置系统限制需要在[systemd配置文件中设置](#systemd)。

## <span id='ulimit'>ulimit</span>

在Linux系统上,可以使用`ulimit`临时改变资源限制。限制通常需要切换到root用户下进行设置。例如，设置打开文件的句柄数限制（ulimit -n）为`65536`，可以按如下做：

```bash
sudo su  #①
ulimit -n 65536 #②
su elasticsearch #③
```

① 切换到root
________________
② 修改最大打开文件句柄数
________________
③ 切换回启动Elasticsearch的`elasticsearch`用户

新的现在将被应用到当前会话。

你可以通过`ulimit -a`查询当前的所有限制参数。

## <span id="limits.conf">/etc/security/limits.conf</span>

在Linux系统，可以通过制定的用户编辑`/etc/security/limits.conf`文件来持久化限制设置。若要设置`elasticsearch`用户打开文件句柄数最大值为`65536`，可以将如下行添加到`limit.conf`文件中：

```js
elasticsearch  -  nofile  65536
```

这种变化知会在用户下次打开一个新的会话时生效。

> 注意
>
> **Ubuntu与`limits.conf`**
> Ubuntu在`init.d`启动时忽略了`limits.conf`文件。要开启`limits.conf`文件，需要编辑`/etc/pam.d/su`，删除下面行的注释：
>
> ```bash
> # session    required   pam_limits.so
> ```

## <span id="sysconfig">Sysconfig file</span>

在使用RPM或Debian系统时，系统设置与环境变量可以通过系统配置文件制定，他们位于：

RPM     /etc/sysconfig/elasticsearch
____________________________________
Debian  /etc/default/elasticsearch

然而，对于使用systemd的系统,系统限制通过[systemd](#systemd)指定。

## <span id="systemd">systemd file</span>

当使用RPM或Debian软件包是使用systemd的系统，系统限制必须通过`systemd`指定。

systemd服务文件（`/usr/lib/systemd/system/elasticsearch.service`）包含的限制将被用作默认值。

如果需要覆盖它，需要添加一个叫`/etc/systemd/system/elasticsearch.service.d/elasticsearch.conf`的文件并在文件中指定任意的变化，就像：

```bash
[Service]
LimitMEMLOCK=infinity
```

## <span id="jvm-options">jvm.options</span>

设置Java虚拟机参数（包括系统属性和JVM参数）的首选方式是通过`jvm.options`文件配置。这个文件在`tar`和`zip`包方式安装时在`config/jvm.options`，在RPM或Debian方式安装时在`/etc/elasticsearch/jvm.options`。这个文件包含了一个用行风格的JVM参数列表，必须要用`-`开头。你可以添加自定义的JVM参数到这个文件或在你的控制系统版本中检查这些配置。

另一种方式是通过设置环境变量`ES_JAVA_OPTS`的方式来添加JAVA虚拟机参数，例如：

```bash
export ES_JAVA_OPTS="$ES_JAVA_OPTS -Djava.io.tmpdir=/path/to/temp/dir"
./bin/elasticsearch
```

在使用RPM或Debian安装包时，`ES_JAVA_OPTS`可以通过[系统配置文件](#systemd)设置。
# 前言

在参考 https://github.com/sucong426/Shadowsocks-Tutorial 使用操作系统为 `OpenCloudOS 9` 的云服务器搭建VPN时遇到了一些困难。这篇文章旨在记录过程中遇到问题的解决方案。如果有一样问题的师傅可以尝试这个方法来解决问题。

该篇文章大部分都沿用了链接中文章的方式进行配置，总结的话就是在自己的服务器上面使用 Shadowsocks 搭建VPN，重复的内容就不再赘述。

## 配置时的问题

### 问题一

使用下面命令下载并且安装Shadowsocks服务的时候可能会遇到一些问题。

```bash
wget --no-check-certificate -O ss.sh https://raw.githubusercontent.com/sucong426/VPN/main/ss.sh
chmod +x ss.sh
sh ss.sh
```

安装的时候我遇到了如下报错，这个报错是由于 `ss.sh` 脚本对操作系统进行了限制，并且 `OpenCloudOS 9` 没有被脚本识别到所以没有安装成功。

```bash
#############################################################
# One click Install Shadowsocks-Python server               #
# Intro: https://teddysun.com/342.html                      #
# Author: Teddysun <i@teddysun.com>                         #
# Github: https://github.com/shadowsocks/shadowsocks        #
#############################################################

[Error] Your OS is not supported. please change OS to CentOS/Debian/Ubuntu and try again.
```

#### 修改脚本

找到 `check_sys` 函数中识别系统的部分。

```python
elif grep -Eqi "OpenCloudOS" /etc/os-release; then
    release="centos"
    systemPackage="yum"
```

将代码修改成下面的形式。

```python
if [[ -f /etc/redhat-release ]]; then
    release="centos"
    systemPackage="yum"
elif grep -Eqi "OpenCloudOS" /etc/os-release; then
    release="centos"
    systemPackage="yum"
elif grep -Eqi "debian|raspbian" /etc/issue; then
    release="debian"
    systemPackage="apt"
...
```

修改脚本之后重新按照链接中文章的方式进行安装和配置。

### 问题二

在按照链接文章完成安装（端口，密码，加密）之后，需要在安全组中增加入站规则，将设置的端口放行。

### 问题三

#### 启动服务

将端口放行之后可以使用下面的命令检查服务进程是否正常在运行。

```bash
ps -ef | grep ssserver
```

之后使用下面的命令检查服务是否正常运行。

```bash
systemctl status shadowsocks
```

如果服务没有正常运行，可以尝试使用下面的命令尝试重新拉起服务。

```bash
/etc/init.d/shadowsocks restart
```

#### 过高 python 版本导致的问题

如果你的服务器python版本在`3.11`或更高时，重启服务的时候会有以下报错。

```bash
Shadowsocks is stopped
Traceback (most recent call last):
  File "/usr/local/bin/ssserver", line 33, in <module>
    sys.exit(load_entry_point('shadowsocks==3.0.0', 'console_scripts', 'ssserver')())
             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/bin/ssserver", line 25, in importlib_load_entry_point
    return next(matches).load()
           ^^^^^^^^^^^^^^^^^^^^
  File "/usr/lib64/python3.11/importlib/metadata/__init__.py", line 202, in load
    module = import_module(match.group('module'))
             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/lib64/python3.11/importlib/__init__.py", line 126, in import_module
    return _bootstrap._gcd_import(name[level:], package, level)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "<frozen importlib._bootstrap>", line 1204, in _gcd_import
  File "<frozen importlib._bootstrap>", line 1176, in _find_and_load
  File "<frozen importlib._bootstrap>", line 1147, in _find_and_load_unlocked
  File "<frozen importlib._bootstrap>", line 690, in _load_unlocked
  File "<frozen importlib._bootstrap_external>", line 940, in exec_module
  File "<frozen importlib._bootstrap>", line 241, in _call_with_frames_removed
  File "/usr/local/lib/python3.11/site-packages/shadowsocks-3.0.0-py3.11.egg/shadowsocks/server.py", line 27, in <module>
    from shadowsocks import shell, daemon, eventloop, tcprelay, udprelay, \
  File "/usr/local/lib/python3.11/site-packages/shadowsocks-3.0.0-py3.11.egg/shadowsocks/udprelay.py", line 71, in <module>
    from shadowsocks import cryptor, eventloop, lru_cache, common, shell
  File "/usr/local/lib/python3.11/site-packages/shadowsocks-3.0.0-py3.11.egg/shadowsocks/lru_cache.py", line 34, in <module>
    class LRUCache(collections.MutableMapping):
                   ^^^^^^^^^^^^^^^^^^^^^^^^^^
AttributeError: module 'collections' has no attribute 'MutableMapping'
Starting Shadowsocks failed
```

这是由于过高的python（高于3.11）版本，不支持 `collections.MutableMapping` 特性导致的安装失败。我们在报错中找到 `lru_cache.py`的地址，对下面的代码进行修改。

```python
class LRUCache(collections.MutableMapping):
```

将该代码修改成下面的形式。

```python
from collections.abc import MutableMapping

class LRUCache(MutableMapping):
```

接着重启服务就可以了。


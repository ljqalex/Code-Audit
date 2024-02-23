## SSRF和伪协议

协议和伪协议：

协议

计算机场景下的协议常常指的是通信协议(网络协议)。网络协议是通信计算机双方必须共同遵从的一组约定。如怎么样建立连接、怎么样互相识别等。只有遵守这个约定，计算机之间才能相互通信交流。

伪协议

伪协议其实算是一个虚概念，就是并不是所有语言、所有产品共用的。往往是某个语言或者某个程序自身，为了解决自身内部的通信需求，自行编制的一个协议。

php里的伪协议：

具体看这里：https://www.php.net/manual/zh/wrappers.php

- [file://](https://www.php.net/manual/zh/wrappers.file.php) — 访问本地文件系统
- [http://](https://www.php.net/manual/zh/wrappers.http.php) — 访问 HTTP(s) 网址
- [ftp://](https://www.php.net/manual/zh/wrappers.ftp.php) — 访问 FTP(s) URLs
- [php://](https://www.php.net/manual/zh/wrappers.php.php) — 访问各个输入/输出流（I/O streams）
- [zlib://](https://www.php.net/manual/zh/wrappers.compression.php) — 压缩流
- [data://](https://www.php.net/manual/zh/wrappers.data.php) — 数据（RFC 2397）
- [glob://](https://www.php.net/manual/zh/wrappers.glob.php) — 查找匹配的文件路径模式
- [phar://](https://www.php.net/manual/zh/wrappers.phar.php) — PHP 归档
- [ssh2://](https://www.php.net/manual/zh/wrappers.ssh2.php) — 安全外壳协议 2
- [rar://](https://www.php.net/manual/zh/wrappers.rar.php) — RAR
- [ogg://](https://www.php.net/manual/zh/wrappers.audio.php) — 音频流
- [expect://](https://www.php.net/manual/zh/wrappers.expect.php) — 处理交互式的流

**php伪协议利用代审通解**

```
跟踪输入点
输入点进入到文件系统操作函数:readfile()、file()、file_get_contents()这种
能够控制参数的开头
```

```php
<?php

$input = $_GET['input'];

file_get_contents($input."qq");  //有缺陷

file_get_contents("qq".$input);  //没有缺陷

?>
```

第一个代码片段中，`file_get_contents($input."qq")`会将用户输入的内容与字符串"qq"合并，然后使用`file_get_contents`函数来读取文件。这种方法存在安全风险，因为用户可以通过在输入中添加恶意内容来访问系统中的任何文件，可能导致信息泄露或文件损坏。这种做法是不安全的，因为用户输入没有经过过滤或验证。

ssrf的发展和起源：

![image-20240223154054739](https://raw.githubusercontent.com/ljqalex/image/main/image-20240223154054739.png)

SSRF(server-side request forgery)为服务端请求伪造，是一种由攻击者形成服务器端发起的安全漏洞。在比较早期的时候，大概是2010年到2017年中的时候，ssrf似乎还不是很盛行，当时我们遇到项目或者可能产生现在常用的ssrf场景时，往往只是利用这个ssrf去进行一个反向代理的作用，当时，ssrf更多还是本地资源的探测和访问、还有内网资源的探测和访问。

ssrf与csrf的差别
ssrf攻击的是服务端或者服务端所在的内网资源，并且帮助我们发起攻击的是服务端。

csrf攻击的是受害者的pc，帮助我们发起攻击的是访问页面的受害者的浏览器

curl支持的ssrf：

curl在不同语言都有插件或者扩展，所以如果我们在代审的时候，遇到某个ssrf漏洞是调用类以curl的模块去进行访问的时候，curl支持的协议均可以搞。

比如php：

```php
<?php
function curl($url)
{
    $ch = curl_init();
    curl_setopt($ch, CURLOPT_URL, $url);
    curl_setopt($ch,CURLOPT_HEADER, O);
    curl_exec($ch);
    curl_close($ch);
    $url = $_GET['url'];
    curl($url);
}
?>
```

ssrf的利用方式：

```php
<?php
$file = $_GET['file'];
$resp = file_get_contents($file);
echo $resp;
?>
```

这个洞很明显，程序员本意可能是获取我们输入指定的一个文件，但是与刚才我们说的伪协议利用代审通解一致的是，我们可以控制file_get_contents函数参数的开头。所以此处我们可以有多个利用，需要特别提出的是:ssrf的利用，其实是利用通信协议对内部组件进行攻击ssrf漏洞的存在，帮我们构造了一个内部环境的代理agent，通过agent，我们可以对内部环境进行窥探，当内部某个组件存在缺陷的时候，我们就可以攻击.

这里有两篇小编觉得很nice的文章：

https://www.freebuf.com/vuls/262047.html

https://book.hacktricks.xyz/pentesting-web/ssrf-server-side-request-forgery

**ssrf转任意文件读取**
利用协议:

```
file   //几乎通用
ldap、zlib、phar、tar、rar //php
jar //java
jar war tar
```

通过ssrf对本地文件进行读取，可以用来做代码审计、或者配置文件读取、密码读取。等等也是一个非常好用的协议。

**ssrf攻击内部应用**
这里的应用特指非web的应用
利用协议:

```
dict
gopher
```

这两个协议也被称之为ssrf中的万金油协议了，因为他们都是封装协议(裸协议)，可以用来封装其他协议。
gopher协议的利用，可以看这里，主要了解他是怎么构成和封装的就可以了:

这是gother协议的利用

https://www.cnblogs.com/h0cksr/p/16189737.html

其实就是通过dict或者gopher协议去构造其他应用的通信协议，与其他应用通信。然后攻击其他应用。
目前国内网络上比较多的被用来搭配ssrf打的内部应用大致有:
redis
fastCGI/php-fpm

ssrf代码审计利用：

再python里面，ssrf并没有那么容易利用：

```python
import urllib.request
url ="dada://127.0.0.1:8080/test/?test=a"
url = "file:///etc/passwd"
info = urllib.request.urlopen(url)
print(info.read())
```

这里有两个协议我们可以利用，一个是fie协议，可以用来读取本地文件。
一个是data协议，比较有趣一点，可以嵌入一些我们的任意输入，但是现在网络上似乎没有什么利用点，这个大家可以研究下

链接：https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Basics_of_HTTP/Data_URLs

---

layout: post
title: "搜狐的localhost漏洞"

---

{% include JB/setup %}

The Tagled Web的第九章提到一个很有意思的localhost漏洞。该漏洞由Tavis Ormandy在2008年首次发现，当时很多知名网站都有该漏洞，包括microsoft，ebay，yahoo等等。事实上这个漏洞危害程度一般，但是最近[Homakov对github的hack(需爬墙)](http://homakov.blogspot.com/2014/02/how-i-hacked-github-again.html)再次证明了多个小漏洞加在一起足以威胁整个系统的安全。

在写本文时经测试该漏洞仍然存在于[搜狐](http://www.sohu.com/)以及[中华网博客](http://blog.china.com/)上。

<!--more-->

localhost漏洞的原理其实很简单，有时候网站的管理员或者开发者会为了方便调试之类的原因向网站中添加localhost子域名，并使其指向127.0.0.1。例如，从下图中可以看到，`ping localhost.sohu.com`得到的地址是127.0.0.1，但是`ping localhost.sina.com.cn`则显示找不到主机（无此漏洞）。

[![localhost漏洞测试]({{ site.url }}/assets/localhost_loophole/1.png)]({{ site.url }}/assets/localhost_loophole/1.png)

但是localhost子域名指向127.0.0.1有什么问题呢？众所周知127.0.0.1是指向本地的一个地址，换句话说用户的浏览器在访问`http://localhost.sohu.com`的时候实际上是将该请求指向了用户本地，但是浏览器却认为它在访问一个`sohu.com`的子域名，因此如果恰好用户的某个本地程序有XSS漏洞就会导致严重的安全问题。

举个最简单的例子，假设用户本地有[一个包含XSS漏洞的程序](https://gist.github.com/eirikrwu/9188993)（这是一个简单的本地XSS漏洞示例程序，该程序监听127.0.0.1:80，并将它接收到HTTP请求中的URL的query部分作为HTTP response返回）。由于用户的浏览器在访问`http://localhost.sohu.com`时认为它在访问一个`sohu.com`的子域名，因此浏览器会将域设置为`*.sohu.com`的cookie全部包含在这个请求中一起发出，另外由于这里搜狐的cookie实际上是没有加httponly标志的（annother simple mistake！），因此可以简单的通过javascript读取出来。因此，如果将以下html代码加入到`http://localhost.sohu.com`的query部分：

	<html>
		<script>
			alert(document.cookie);
		</script>
	</html>

得到的URL `http://localhost.sohu.com/?%3Chtml%3E%3Cscript%3Ealert(document.cookie);%3C/script%3E%3C/html%3E`即可显示用户在搜狐上的所有cookie，如下图所示。

[![使用localhost漏洞获取cookie]({{ site.url }}/assets/localhost_loophole/2.png)]({{ site.url }}/assets/localhost_loophole/2.png)

这里如果将alert换成`window.location.href=`再指定一个接受地址的话即可劫持用户的会话（这里假设得到的cookie中包含搜狐的用户身份认证cookie，根据得到的cookie来看应该是没有问题的，但是并没有进一步进行测试验证）。

以上就是localhost漏洞的基本原理，该漏洞的威胁度其实一般，如果用户本地没有有XSS漏洞的程序的话就没有什么大问题。最后还有一个有趣的地方是，上文所提到的[包含XSS漏洞的示例程序](https://gist.github.com/eirikrwu/9188993)经过编译后在楼主的机器上使用avast扫描没有发现任何问题（其实也不应该有问题，仅仅是个绑定127.0.0.1并且处理http请求的程序而已），但是若干个这样的小漏洞结合起来往往就会带来致命的后果。
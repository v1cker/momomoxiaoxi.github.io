---
title: 网络流量分析与信息还原
time: 2016.01.26 21:47:00
layout: post
catalog: true
tags:
- Security
- Wireshark
excerpt: 网络嗅探截获的是在通过封包过程组装的二进制格式原始报文内容。若我们要获取其中包含的信息，只需要依据TCP/IP协议栈的协议规范，依次还原出数据包在各个协议层上的协议格式及其内容，以及在应用层传输的实际数据。这篇日志，我将展示通过wireshark进行Http协议分析，并尝试还原图片文件。
    


---

# 流量分析与信息还原

### 实验背景：
OWASP节点和Seed Ubuntu节点之间正常通行时数据包将不会经过Kali节点。 攻击节点Kali将通过wireshark抓包监听，从而截获Seed Ubuntu和OWASP节点之间的数据包，并可对数据包内容进行分析和还原。### 实验内容及步骤1.	在Kali节点的终端打开Wireshark
	root@kali:~# wireshark
	![image](http://momomoxiaoxi.com/img/post/wireshark/1.png)	Wireshark初始界面2.	选择监听网卡，并开始监听
	Capture->Options,选择网卡eth0，并开始监听：
	![image](http://momomoxiaoxi.com/img/post/wireshark/2.png)
	Wireshark 设置界面3.	访问www.baidu.com，获取数据
	打开浏览器，访问百度，获得数据，并使用过滤器过滤出http数据：	![image](http://momomoxiaoxi.com/img/post/wireshark/3.png)
	HTTP协议访问百度	**整个过程分析：**
	在浏览器中输入www.baidu.com，敲击回车的过程中，浏览器向DNS服务器请求解析www.baidu.com的IP地址。域名系统DNS解析出百度服务器的IP地址为103.235.46.39.（这里百度有两个ip，一个是103.235.46.39，还一个63.217.158.168，我们主要分析与103.235.46.39的行为。服务器具有多个ip是主要用于分压与抗攻击）然后，浏览器与对应服务器建立TCP连接(三次握手协议，具体可查看前面几个包，服务器端的IP地址为103.235.46.39，端口是80)。然后浏览器发出取文件命令：GET / HTTP/1.1。服务器给出响应把文件(text/html)发送给浏览器，浏览器显示text/html中的所有文本。浏览器下载网页文本内容，网页文本中标记着图片、CSS文件和Flash等等。在这次访问过程中,百度主页还包括百度logo图片和其他一些内容，浏览器分析出这些内容后，开启多线程对这些内容进行下载，分别向服务器发送请求报文，服务器接收到内容后根据HTTP协议发送响应报文。所有的内容下载完毕时候,浏览器会显示全部内容，一个完整的百度主页就这样打开了。
4.  分析GET数据包
    为了防止额外包干扰，我们修改过滤条件，使其只显示本机与百度主ip的http包：	![image](http://momomoxiaoxi.com/img/post/wireshark/4.png)
    本机与百度的通信包	从上图可以看到，本机ip地址是172.16.137.129，百度的ip地址是103.235.46.39。	首先，本机向百度代理服务器发送HTTP请求报文，请求服务器发送文本请求：	![image](http://momomoxiaoxi.com/img/post/wireshark/5.png)	GET包信息
    代码分析：	
    	GET / HTTP/1.1\r\n         							   //请求目标		Host： www.baidu.com\r\n                              //目标所在的主机		User-Agent： Mozilla/5.0 (X11;Linux x86_64; rv;38.0) Gecko/20100101 Firefox/38.0 Iceweasel/38.2.1\r\n      //用户代理，浏览器的类型是Iceweasel浏览器；括号内是相关解释		Accept： text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8\r\n \\接受数据信息   		Accept-Language： en-US,en;q=0.5\r\n          //语言英文		Accept-Encoding： gzip,deflate\r\n          //可接受编码，文件格式		Connection： keep-alive                                  //激活连接
    5.	分析返回包	![image](http://momomoxiaoxi.com/img/post/wireshark/6.png)
        返回包信息整体分析：200状态码，表示百度服务器成功接收到我们的请求报文，它向本地（172.16.137.129）发送响应报文，把文件发送给浏览器。	代码分析：		HTTP/1.1 200 OK\r\n							// 状态码200，响应正常    	Date: Sun, 25 Jan 2016 02:58:58 GMT\r\n 		//响应信息创建的时间    	Content-Type: text/html; charset=utf-8\r\n		//返回内容类型和编码类型    	Transfer-Encoding: chunked\r\n					//采用分块传输编码    	Connection: Keep-Alive\r\n						//保持激活状态    	Vary: Accept-Encoding\r\n		//告诉代理服务器缓存两种版本的资源：压缩和非压缩，这有助于避免一些公共代理不能正确地检测Content-Encoding标头的问题。		Set-Cookie: BAIDUID=E7851FBF24BD2381E3129B388460A89E:FG=1; expires=Thu, 31-Dec-37 23:55:55 GMT; max-age=2147483647; path=/; domain=.baidu.com\r\n    	Set-Cookie: BIDUPSID=E7851FBF24BD2381E3129B388460A89E; expires=Thu, 31-Dec-37 23:55:55 GMT; max-age=2147483647; path=/; domain=.baidu.com\r\n    	Set-Cookie: PSTM=1453690738; expires=Thu, 31-Dec-37 23:55:55 GMT; max-age=2147483647; path=/; domain=.baidu.com\r\n    	Set-Cookie: BDSVRTM=9; path=/\r\n    	Set-Cookie: BD_HOME=0; path=/\r\n    	Set-Cookie: H_PS_PSSID=18880_17745_1458_18879_12824_18965_18777_18134_17001_18781_17073_15311_12313; path=/; domain=.baidu.com\r\n			//设置cookie值		P3P: CP=" OTI DSP COR IVA OUR IND COM "\r\n    		Cache-Control: private\r\n			//高速缓存设置为private，具体作用依据不同的重新浏览方式会有所不用    		Cxy_all: baidu+da9670a5f461ab5cf1a7ed60593445da\r\n    		Expires: Mon, 25 Jan 2016 02:58:58 GMT\r\n		//内容过期时间    	X-Powered-By: HPHP\r\n    	Server: BWS/1.1\r\n    	X-UA-Compatible: IE=Edge,chrome=1\r\n    	BDPAGETYPE: 1\r\n    	BDQID: 0xf35567a50011f638\r\n    	BDUSERID: 0\r\n    	Content-Encoding: gzip\r\n    	\r\n    	[HTTP response 1/4]    	[Time since request: 0.360694000 seconds]    	[Request in frame: 10]    	[Next request in frame: 26]    	[Next response in frame: 83]    	HTTP chunked response    	Content-encoded entity body (gzip): 26639 bytes -> 98202 bytes		Line-based text data: text/html6.		分析请求获取百度logo包
        ![image](http://momomoxiaoxi.com/img/post/wireshark/7.png)GET百度logo包信息	在前面的包可以看到，还有需要下载的内容，该包为本地服务器再次发送请求报文，向百度服务器请求百度logo文件。	代码分析：    	GET /img/bd_logo1.png HTTP/1.1\r\n						//请求获取bd_logo1.png    	Host: www.baidu.com\r\n								//目标所在主机    	User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:38.0) Gecko/20100101 Firefox/38.0 Iceweasel/38.2.1\r\n											//浏览器类型    	Accept: image/png,image/*;q=0.8,*/*;q=0.5\r\n			//返回包类型    	Accept-Language: en-US,en;q=0.5\r\n						//语言    	Accept-Encoding: gzip, deflate\r\n					//可接受编码，文件格式    	Referer: http://www.baidu.com/\r\n	//告诉服务器该请求从哪个页面链接而来    	Cookie: BAIDUID=E7851FBF24BD2381E3129B388460A89E:FG=1; BIDUPSID=E7851FBF24BD2381E3129B388460A89E; PSTM=1453690738; BDSVRTM=9; BD_HOME=0; H_PS_PSSID=18880_17745_1458_18879_12824_18965_18777_18134_17001_18781_17073_15311_12313\r\n    	Connection: keep-alive\r\n			//保持激活    	\r\n    	[Full request URI: http://www.baidu.com/img/bd_logo1.png]    	[HTTP request 2/4]    	[Prev request in frame: 10]    	[Response in frame: 83]    	[Next request in frame: 223]
        7.分析png返回包
        ![image](http://momomoxiaoxi.com/img/post/wireshark/8.png)
        百度logo返回包信息状态行显示的结果表明百度服务器成功接收到本地发送的请求报文，相应的的向本地发出响应报文并把图像文件发送给了本地。
        响应报文中有关于图片的一些信息记录。代码分析：		Hypertext Transfer Protocol    	HTTP/1.1 200 OK\r\n						//状态码200，响应正常    		Date: Mon, 25 Jan 2016 02:58:58 GMT\r\n		//响应创建时间    		Server: Apache\r\n								//服务器    		Last-Modified: Wed, 03 Sep 2014 10:00:27 GMT\r\n		//最近修改时间    		ETag: "1ec5-502264e2ae4c0"\r\n					//被请求变量的实体值    	Accept-Ranges: bytes\r\n						//接受数据格式    	Content-Length: 7877\r\n						//数据长度    	Cache-Control: max-age=315360000\r\n			//缓存控制    	Expires: Thu, 22 Jan 2026 02:58:58 GMT\r\n			//过期时间    	Connection: Keep-Alive\r\n						//保持激活    	Content-Type: image/png\r\n						//内容类型为图片    	\r\n    	[HTTP response 2/4]    	[Time since request: 0.608656000 seconds]					//响应时间    	[Prev request in frame: 10]    	[Prev response in frame: 24]    	[Request in frame: 26]    	[Next request in frame: 223]    	[Next response in frame: 229]		Portable Network Graphics
        8.复原png图片
        找到png的返回包，右击Follow TCP Stream，可以看到下面的图![image](http://momomoxiaoxi.com/img/post/wireshark/9.png)	TCP Stream	
        找到png部分，易知png的数据肯定是百度代理服务器传输过来的，所以我们保存下面的文件，以二进制保存，保存为logo.bin ![image](http://momomoxiaoxi.com/img/post/wireshark/10.png)
         保存文件 在TCP Stream中，我们可以看到，png的原始文件在Content-Type: image/png加两个回车符后面。回车符用16进制表示就是0D 0A。所以，我们一般找到0D 0A 0D 0A，它们后面部分就是图片的开始，这样你就可以手动去寻找图片。同理，也可以找到文件的末尾信息，最后截出整个图片信息。
         ![image](http://momomoxiaoxi.com/img/post/wireshark/11.png)PNG信息 下面，我们主要演示用binwalk来分解出图片：![image](http://momomoxiaoxi.com/img/post/wireshark/12.png)	
        binwalk分析
        可以看到在这个二进制文件中，确实存在一个png图片。使用binwalk提取文件binwalk --dd='png'  logo.bin可以看到，在当前目录下多了一个文件夹，里面就是我们要提取的图片文件：
        ![image](http://momomoxiaoxi.com/img/post/wireshark/13.png)提取后的图片文件### 问题说明1.抓包时看到HTTP 304状态码，无法抓到图片包。304状态码表示页面未修改，即自从上次请求后，请求的网页未修改过。服务器返回此响应时，不会返回网页内容。此时，你需要刷新页面或重新开启浏览器，强制页面内容进行刷新。如果还不行，就清理缓存，重新访问。
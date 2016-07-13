---
layout: post
title:  "github page博客里添加多说评论插件"
date:   2016-06-05 15:25:20 +0700
categories: [blog]
---

**摘要:**  

* **由于现在我这个博客原来用的是DISQUS评论插件，那全是全球的，有些关联被墙**
* **所以我把评论插件换成国内的多说**



* 登录多说创建站点  

 [多说站点创建地址](http://duoshuo.com/create-site/)  

* 创建多说站点，填写信息如下图：  

![duoshuo](/static/img/myimg/duoshuo1.png)

* 填写完信息后，点击创建，生成代码，如下图：  

![duoshuo](/static/img/myimg/duoshuo2.png)  

* 上图代码有三处地方需要修改，代码中已经表示出来了，可以参照我的代码：  
  
{% highlight html %} 
<!-- 多说评论框 start -->
	<div class="ds-thread"
  data-thread-key="{{ site.url }}_{{ page.title }}"data-title="{{ page.title }}"data-url="{{ site.url }}"></div>
<!-- 多说评论框 end -->
<!-- 多说公共JS代码 start (一个网页只需插入一次) -->
<script type="text/javascript">
var duoshuoQuery = {short_name:"galengao"};
	(function() {
		var ds = document.createElement('script');
		ds.type = 'text/javascript';ds.async = true;
		ds.src = (document.location.protocol == 'https:' ? 'https:' : 'http:') + '//static.duoshuo.com/embed.js';
		ds.charset = 'UTF-8';
		(document.getElementsByTagName('head')[0]
		 || document.getElementsByTagName('body')[0]).appendChild(ds);
	})();
	</script>
<!-- 多说公共JS代码 end -->
{% endhighlight %}

* 将此代码粘贴到需要的位置,至于放在什么文件什么地方,取决于你所使用的模板。  

一般位置在`_layouts/post.html`    
或者`_layouts/default.html`  

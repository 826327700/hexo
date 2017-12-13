---
title: http请求与cors跨域的实践记录
date: 2017-12-13 21:27:36
categories: "node" 
tags: 
     - javascript
     - node
---
###起因    
最近在前后端分离开发的项目中，遇到了跨域请求的问题，当时首选的是cors跨域解决方案。在以往的项目中也是大部分使用此方案进行跨域请求，但以往似乎都很顺利，并没有遇到最近遇到的问题。最近的这个项目，严格按照Restful API的风格，并且要求使用content-type为application/json的编码格式，于是就出问题了。
###问题
此项目后端要求http请求时，设置content-type为application/json编码格式，由此出现了如下问题：（不方便直接贴项目源码，故单独写了一个客户端和服务端的demo）
![image.png](http://upload-images.jianshu.io/upload_images/6651371-509fde6333a06512.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image.png](http://upload-images.jianshu.io/upload_images/6651371-65122c04eff93d14.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
###分析
在网上搜索了一番之后，才深入了解到关于cors通信的详细规定。
CORS是一个W3C标准，全称是"跨域资源共享"（Cross-origin resource sharing）。
它允许浏览器向跨源服务器，发出XMLHttpRequest请求，从而克服了AJAX只能同源使用的限制。
整个CORS通信过程，都是浏览器自动完成，不需要用户参与。对于开发者来说，CORS通信与同源的AJAX通信没有差别，代码完全一样。浏览器一旦发现AJAX请求跨源，就会自动添加一些附加的头信息，有时还会多出一次附加的请求，但用户不会有感觉。
因此，实现CORS通信的关键是服务器。只要服务器实现了CORS接口，就可以跨源通信。

浏览器将CORS请求分成两类：简单请求（simple request）和非简单请求（not-so-simple request）。
只要同时满足以下两大条件，就属于简单请求。
（1) 请求方法是以下三种方法之一：
        HEAD
        GET
        POST
（2）HTTP的头信息不超出以下几种字段：
       Accept
       Accept-Language
       Content-Language
       Last-Event-ID
       Content-Type：只限于三个值application/x-www-form-urlencoded、multipart/form-data、text/plain
凡是不同时满足上面两个条件，就属于非简单请求。
浏览器对这两种请求的处理，是不一样的。

非简单请求是那种对服务器有特殊要求的请求，比如请求方法是PUT或DELETE，或者Content-Type字段的类型是application/json。
非简单请求的CORS请求，会在正式通信之前，增加一次HTTP查询请求，称为"预检"请求（preflight）。
浏览器先询问服务器，当前网页所在的域名是否在服务器的许可名单之中，以及可以使用哪些HTTP动词和头信息字段。只有得到肯定答复，浏览器才会发出正式的XMLHttpRequest请求，否则就报错。

**到这里大致可以确定，出现问题是因为使用了非简单请求引起的。**

回顾一下，后端在开启cors跨域时一般设置如下（node示例，实际项目为java）
![image.png](http://upload-images.jianshu.io/upload_images/6651371-69bc22b2cb0ee42e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
app.all('*',function(req, res, next) {
    res.setHeader('Access-Control-Allow-Origin', '*');//或者是指定的ip
    res.setHeader('Access-Control-Allow-Methods', 'GET, POST');
    next();
});
```
第一个是设置允许跨域请求来源，*号为允许所有来源。
第二个是设置允许跨域请求方法，一般简单请求就是get和post。

**问题点一、**如果严格遵照Restful API的风格，除了GET和POST，还有PUT、DELETE等动词性的请求方法，那么就属于非简单请求了。此时：
![image.png](http://upload-images.jianshu.io/upload_images/6651371-7f2e4437546c51b5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
app.all('*',function(req, res, next) {
    res.setHeader('Access-Control-Allow-Origin', '*');
    res.setHeader('Access-Control-Allow-Methods', 'GET, POST,PUT,DELETE....');//把需要使用到的请求方法加上
    next();
});
```
**问题点二、**如果需要设置content-type：application/json，同样属于非简单请求了，因此需要：
![image.png](http://upload-images.jianshu.io/upload_images/6651371-b52c30e5c16e4a2f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
app.all('*',function(req, res, next) {
    res.setHeader('Access-Control-Allow-Origin', '*');
    res.setHeader('Access-Control-Allow-Methods', 'GET, POST');
    res.setHeader('Access-Control-Allow-Headers', 'content-type');//设置允许自定义的头信息
    next();
});
```
第三个设置允许自定义设置头信息content-type，此时方可在http请求中设置content-type为application/json，并且可以顺利进行请求了。

既然上面提到过非简单请求会增加一次http请求，用于预先检测合法性，来看一下：
![image.png](http://upload-images.jianshu.io/upload_images/6651371-a09f105cd608f45f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
1.预检测请求方法为OPTIONS
2.请求设置头信息为content-type
3.本次正式请求的方法为POST
再去看看Response Headers的内容
Access-Control-Allow-Headers:content-type
Access-Control-Allow-Methods:GET, POST
服务器允许设置的头信息为content-type，允许使用的请求方法为GET, POST，所以预请求完全合格，故可以顺利进行正式请求：
![image.png](http://upload-images.jianshu.io/upload_images/6651371-6bf4aa15a30da288.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
正式请求方法为POST，content-type为客户端设置的application/json，到此，大功告成。
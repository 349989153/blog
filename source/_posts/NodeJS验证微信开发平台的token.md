---
title: NodeJS验证微信开发平台的token
date: 2018-04-10 14:37:23
tags: [js,node,微信]
categories: 技术
---

<!-- more -->
---
如果想在微信公众号上让用户体验更多的功能，而不仅仅是让公众号像机器人一样收发有限的信息，你就要成为微信平台开发者，自主开发web app。

成为微信平台开发者的第一步，是配置一台web服务器。
第二步，是配置相关的参数，通过微信的开发者验证，并且把你的服务器地址告知微信服务器。这里单讲第二步。

详细的配置步骤写在这里：[微信公众平台开发者文档-接入指南][1]

我个人提炼一下这个过程的话，就是微信服务器生成了一系列随机字符，放在url的query里，然后根据你写的服务器地址，发送request。而你的服务器在收到了request之后，分别取出query的各项参数，按照他的规则拼接啦，哈希变化，然后再返回（response）一个值给微信服务器。微信服务器收到之后，如果等于服务器本身设定的那个值，就通过验证。

后端语言有很多，php，python，node，还有windows平台的，我们这里用node。

下面是node源码，注意，该源码不是我原创，而是[csdn博客-NoGrief的博客][2] 的代码调试修改而来。

```javascript
var http = require("http");
var url = require("url");
var crypto = require("crypto");

function sha1(str){
  var md5sum = crypto.createHash("sha1");
  md5sum.update(str);
  str = md5sum.digest("hex");
  return str;
}

function validateToken(req,res){
  var query = url.parse(req.url,true).query;
  //console.log("*** URL:" + req.url);
  //console.log(query);
  var signature = query.signature;
  var echostr = query.echostr;
  var timestamp = query['timestamp'];
  var nonce = query.nonce;
  var oriArray = new Array();
  oriArray[0] = nonce;
  oriArray[1] = timestamp;
  oriArray[2] = "*********";//这里是你在微信开发者中心页面里填的token，而不是****
  oriArray.sort();
  var original = oriArray.join('');
  console.log("Original str : " + original);
  console.log("Signature : " + signature );
  var scyptoString = sha1(original);
  if(signature == scyptoString){
    res.end(echostr);
    console.log("Confirm and send echo back");
  }else {
    res.end("false");
    console.log("Failed!");
  }
}


var webSvr = http.createServer(validateToken);
webSvr.listen(8000,function(){
  console.log("Start validate");
});
```

上面代码保存为tokenValidate.js或者任何你喜欢的名字。

你也许会问，为啥监听的是8000端口而不是80端口？
因为ubuntu下使用80端口很麻烦而且有安全问题，所以我上网查了资料，把8000端口的监听也改在80端口，这样一来，8000端口的信息会自动转到80端口去。
这是资料：[StackOverFlow-Best practices when running Node.js with port 80(Ubuntu/Lincode)][3]

然后在微信开发者中心里点提交，就可以了。


  [1]: http://mp.weixin.qq.com/wiki/17/2d4265491f12608cd170a95559800f2d.html
  [2]: http://blog.csdn.net/nogrief/article/details/9774773
  [3]: http://stackoverflow.com/questions/16573668/best-practices-when-running-node-js-with-port-80-ubuntu-linode
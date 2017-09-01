---
title: Atom 编写 markdown 上传图片
date: 2017-07-23 15:34:13
tags: markdown
categories: Markdown
---

因为对markdown比较感兴趣,平时的文档编写都是用markdown来完成.以前是用csdn的博客编辑器,支持图片上传,预览. 最近在公司的文档也开始用markdown来写了,就会遇到一个问题：如何上传图片？

最开始我不知道图床啥的,就是自己先把图片上传到 github ,然后再手动引用连接到文章中,这样的也能实现markdown的图片引用. 但是不够优雅,而且很麻烦.上传一张图片有4到5步操作.

然后发现了 Atom编辑器的两个插件,使用 7牛 作为图床可以实现： qq截图,然后直接在 Atom 的编辑页面 `ctrl+v` 粘贴, 弹出上传图片提示，会让你填写一个 **title** ，点击enter会自动构建一条markdown图片语句。这样上传图片就完成了.

效果如下：
![七牛云图床,一键上传图片](http://otiwf3ulm.bkt.clouddn.com/57ba925c75d800e6bfaeb1c3ddbf171d.png)

这个就是我一键上传的图片,是不是很简单! 下面我就给大家介绍实现的方法:

### 第一步, 注册7牛云


7牛云的注册,新建空间就不过多介绍了.

吴同学博客: [使用七牛作为github博客的图床
](http://wuxiaolong.me/2014/10/30/qiniu-photo-bed/)

佚名大神博客: [如何使用七牛](https://xbotao.github.io/hexo-admin-qiniu/2017/02/08/qiniu.html)

相信大家可以搞定的.

### 第二步, 下载 markdown-assistant,qiniu-uploader 插件

![chajian](http://otiwf3ulm.bkt.clouddn.com/d9153249393eb1c106781ed1c6866207.png)

安装好插件后还需要对它们进行设置:

设置markdown-assistant的时候发现，会让你填一个上传插件，而且默认已经帮你填好了qiniu-uploader，其实你也就什么都不用设置了。

如下图:

![markdown-assistant](http://otiwf3ulm.bkt.clouddn.com/6189cfff45711be2724861ae433c548d.png)

设置qiniu-uploader，主要设置四个参数：

![qiniu-uploader](http://otiwf3ulm.bkt.clouddn.com/860bd5505cf4ca4a196672eef5f10b26.png)

### qiniu-uploader 使用报错解决方法

我自己在使用 `qiniu-uploader` 时Atom提示错误,如下图:

![qiniu-error](http://otiwf3ulm.bkt.clouddn.com/da79c8140fd97dc4248e3d616d06b3a9.png)

可能你们也会遇到这个问题,
[解决方案地址](https://github.com/knightli/qiniu-uploader/issues/6)

这里我整理了一下步骤:

 1. 先找到 Atom 的配置文件,再找到插件的安装路径. 我的电脑是Windows系统,路径是是这样的 : `C:\Users\Administrator\.atom\packages\qiniu-uploader\node_modules\qiniu\qiniu`

 2. 替换 `\qiniu` 文件夹下面的三个 .js 文件: `io.js`,`rpc.js`,`zone.js`, 然后就ok了

为了方便大家更换这三个文件,我就贴出来了:

os.js
```
var conf = require('./conf');
var util = require('./util');
var rpc = require('./rpc');
var fs = require('fs');
var getCrc32 = require('crc32');
var url = require('url');
var mime = require('mime');
var Readable = require('stream').Readable;
var formstream = require('formstream');
var urllib = require('urllib');
var zone = require('./zone');

exports.UNDEFINED_KEY = '?'
exports.PutExtra = PutExtra;
exports.PutRet = PutRet;
exports.put = put;
exports.putWithoutKey = putWithoutKey;
exports.putFile = putFile;
exports.putReadable = putReadable;
exports.putFileWithoutKey = putFileWithoutKey;

// @gist PutExtra
function PutExtra(params, mimeType, crc32, checkCrc) {
  this.params = params || {};
  this.mimeType = mimeType || null;
  this.crc32 = crc32 || null;
  this.checkCrc = checkCrc || 0;
}
// @endgist

function PutRet(hash, key) {
  this.hash = hash || null;
  this.key = key || null;
}

// onret: callback function instead of ret
function putReadable (uptoken, key, rs, extra, onret) {
  if (!extra) {
    extra = new PutExtra();
  }
  if (!extra.mimeType) {
    extra.mimeType = 'application/octet-stream';
  }

  if (!key) {
    key = exports.UNDEFINED_KEY;
  }

  rs.on("error", function (err) {
      onret({code: -1, error: err.toString()}, {});
  });


  // 设置上传域名
  // zone.up_host(uptoken, conf);
  zone.up_host_async(uptoken, conf, function() {
      var form = getMultipart(uptoken, key, rs, extra);
      return rpc.postMultipart(conf.UP_HOST, form, onret);
  });

}


function put(uptoken, key, body, extra, onret) {
  var rs = new Readable();
  rs.push(body);
  rs.push(null);

  if (!extra) {
    extra = new PutExtra();
  }
  if (extra.checkCrc == 1) {
    var bodyCrc32 = getCrc32(body);
    extra.crc32 = '' + parseInt(bodyCrc32, 16);
  } else if (extra.checkCrc == 2 && extra.crc32) {
    extra.crc32 = '' + extra.crc32
  }
  return putReadable(uptoken, key, rs, extra, onret)
}

function putWithoutKey(uptoken, body, extra, onret) {
  return put(uptoken, null, body, extra, onret);
}

function getMultipart(uptoken, key, rs, extra) {

  var form = formstream();

  form.field('token', uptoken);
  if (key != exports.UNDEFINED_KEY) {
    form.field('key', key);
  }
  form.stream('file', rs, key, extra.mimeType);

  if (extra.crc32) {
    form.field('crc32', extra.crc32);
  }

  for (var k in extra.params) {
    form.field(k, extra.params[k]);
  }

  return form;
}

function putFile(uptoken, key, loadFile, extra, onret) {

  var rs = fs.createReadStream(loadFile);

  if (!extra) {
    extra = new PutExtra();
  }
  if (extra.checkCrc == 1) {
    var fileCrc32 = getCrc32(fs.readFileSync(loadFile));
    extra.crc32 = '' + parseInt(fileCrc32, 16);
  } else if (extra.checkCrc == 2 && extra.crc32) {
    extra.crc32 = '' + extra.crc32
  }
  if (!extra.mimeType) {
    extra.mimeType = mime.lookup(loadFile);
  }

  return putReadable(uptoken, key, rs, extra, onret);
}

function putFileWithoutKey(uptoken, loadFile, extra, onret) {
  return putFile(uptoken, null, loadFile, extra, onret);
}

```

rpc.js
```
var urllib = require('urllib');
var util = require('./util');
var conf = require('./conf');

exports.postMultipart = postMultipart;
exports.postWithForm = postWithForm;
exports.postWithoutForm = postWithoutForm;

function postMultipart(uri, form, onret) {
  return post(uri, form, form.headers(), onret);
}

function postWithForm(uri, form, token, onret) {
  var headers = {
    'Content-Type': 'application/x-www-form-urlencoded'
  };
  if (token) {
    headers['Authorization'] = token;
  }
  return post(uri, form, headers, onret);
}

function postWithoutForm(uri, token, onret) {
  var headers = {
    'Content-Type': 'application/x-www-form-urlencoded',
  };
  if (token) {
    headers['Authorization'] = token;
  }
  return post(uri, null, headers, onret);
}

function post(uri, form, headers, onresp) {
  headers = headers || {};
  headers['User-Agent'] = headers['User-Agent'] || conf.USER_AGENT;


  var data = {
    headers: headers,
    method: 'POST',
    dataType: 'json',
    timeout: conf.RPC_TIMEOUT,
  };
  if (Buffer.isBuffer(form) || typeof form === 'string') {
    data.content = form;
  } else if (form) {
    data.stream = form;
  } else {
    data.headers['Content-Length'] = 0;
  };

  var req = urllib.request(uri, data, function(err, result, res) {
    var rerr = null;
    if (err || Math.floor(res.statusCode/100) !== 2) {
      rerr = {code: res&&res.statusCode||-1, error: err||result&&result.error||''};
    }
    onresp(rerr, result, res);
  });

  return req;
}
```

zone.js
```
// var request = require('urllib-sync').request;
var urllib = require('urllib');
var util = require('./util');
//conf 为全局变量
exports.up_host = function (uptoken, conf){

    var version = process.versions;
    var num = version.node.split(".")[0];

    // node 版本号低于 1.0.0, 使用默认域名上传，可以在conf中指定上传域名
    if(num < 1 ){
        conf.AUTOZONE = false;
    }

    if(!conf.AUTOZONE){
        return;
    }

    var ak = uptoken.toString().split(":")[0];
    var tokenPolicy = uptoken.toString().split(":")[2];
    var tokenPolicyStr = new Buffer(tokenPolicy, 'base64').toString();
    var json_tokenPolicyStr = JSON.parse(tokenPolicyStr);

    var scope = json_tokenPolicyStr.scope;
    var bucket = scope.toString().split(":")[0];

    // bucket 相同，上传域名仍在过期时间内
    if((new Date().getTime() < conf.EXPIRE) && bucket == conf.BUCKET){
        return;
    }

    //记录bucket名
    conf.BUCKET = bucket;

    var request_url = 'http://uc.qbox.me/v1/query?ak='+ ak + '&bucket=' + bucket;

    var res = request('http://uc.qbox.me/v1/query?ak='+ ak + '&bucket=' + bucket, {
      'headers': {
        'Content-Type': 'application/json'
      }
    });

    if(res.statusCode == 200){

        var json_str = JSON.parse(res.body.toString());

        //判断设置使用的协议, 默认使用http
        if(conf.SCHEME == 'http'){
            conf.UP_HOST = json_str.http.up[1];
        }else{
            conf.UP_HOST = json_str.https.up[0];
        }

        conf.EXPIRE = 86400 + new Date().getTime();

    }else{
        var err = new Error('Server responded with status code ' + res.statusCode + ':\n' + res.body.toString());
        err.statusCode = res.statusCode;
        err.headers = res.headers;
        err.body = res.body;
        throw err;
    }



}



exports.up_host_async = function (uptoken, conf, callback){

    var version = process.versions;
    var num = version.node.split(".")[0];

    // node 版本号低于 1.0.0, 使用默认域名上传，可以在conf中指定上传域名
    if(num < 1 ){
        conf.AUTOZONE = false;
    }

    if(!conf.AUTOZONE){
        callback();
        return;
    }

    var ak = uptoken.toString().split(":")[0];
    var tokenPolicy = uptoken.toString().split(":")[2];
    var tokenPolicyStr = new Buffer(tokenPolicy, 'base64').toString();
    var json_tokenPolicyStr = JSON.parse(tokenPolicyStr);

    var scope = json_tokenPolicyStr.scope;
    var bucket = scope.toString().split(":")[0];

    // bucket 相同，上传域名仍在过期时间内
    if((new Date().getTime() < conf.EXPIRE) && bucket == conf.BUCKET){
        callback();
        return;
    }

    //记录bucket名
    conf.BUCKET = bucket;
    var request_url = 'http://uc.qbox.me/v1/query?ak='+ ak + '&bucket=' + bucket;
    var data = {
      contentType: 'application/json',
      method: 'GET',
    };


    var req = urllib.request(request_url, data, function(err, result, res) {
      // console.log(result);
      if(res.statusCode == 200){
          // console.log(result);
          var json_str = JSON.parse(result.toString());
          // console.log(json_str);
          //判断设置使用的协议, 默认使用http
          if(conf.SCHEME == 'http'){
              conf.UP_HOST = json_str.http.up[1];
          }else{
              conf.UP_HOST = json_str.https.up[0];
          }

          conf.EXPIRE = 86400 + new Date().getTime();
          callback();
          return;
      }else{
          var err = new Error('Server responded with status code ' + res.statusCode + ':\n' + res.body.toString());
          err.statusCode = res.statusCode;
          err.headers = res.headers;
          err.body = res.body;
          throw err;
          callback();
      }
    });

    return;


}
```

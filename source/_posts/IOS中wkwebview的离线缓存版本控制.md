---
title: IOS中wkwebview的离线缓存版本控制
date: 2017-09-22 15:21:24
tags: [javascript,H5,IOS,WKWebview]
categories: IOS
---

我们IOS主app下面的一个内容模块是H5页面，H5页面首次打开的时候需要等待网络加载，用户体验会比原生的差很多。但是这个H5站点暂时是不会用其他方式再实现一次了。所以先从WEB容器上面做手脚。 
WKWebview本身是有缓存机制的，但是第一次加载的时候难免需要从网络请求。然后就有了做个离线包的想法。 
原理很简单，在IOS打包的时候，直接把静态资源压缩，将压缩包放进App安装包中。这样第一次打开就不用从网络请求了。
<!-- more -->
### 但是具体实现起来，还是有几个问题要考虑的: 

* 缓存规则 
  哪些文件需要缓存在本地，用本地的资源替换，这需要有个配置。 
* 资源的替换方式 
  需要监听网络请求，把符合缓存规则的请求拦截，用本地资源做替换。
* 离线包版本问题  
  如果用户app的版本打包提审后，线上H5的又有了更新，那离线包的资源就比线上的老，如果直接用离线包来展示，可能会出现问题。 
  当发现离线包比较旧的时候，就需要重新从网络请求资源。但如果所有资源都重新从网络请求那就基本相当于离线包失效了。 
  如果一个2M的离线包，只更新了10K的内容，应该就可以采用创建diff的方式来打更新补丁。这样就可以利用起离线包资源。 
* 版本更新逻辑 
  这主要是产品逻辑问题。 
  用户进入模块的时候，需要请求版本信息接口，如果有新的版本，是先展示旧版本，后台静默更新，下次打开页面时自动更新;还是提示用户更新，主动更新后重启。 
  如果用旧的离线包来展示，就需要做好数据接口的版本控制，这个对后端要求较高；或者保证接口的更新必须兼容旧版本前端代码，这个在原生app上面应该都是可以满足的。 
  更细致的，可以在资源版本信息上面加上控制字段，判断哪些版本是不必须立即更新的。以及加上预下载机制，在版本上线前把资源提前下发到客户端。  
* 离线包生效机制
  首次进入模块时，离线包是在资源目录中，还没有存在沙盒文件系统，需要从资源包中解析并取出。APP更新时，已经有了本地离线包，但是也还需要与线上的离线包和APP的安装包作比较，这里得注意他们的的比较逻辑。 
* 缓存文件的清理机制
  下载的缓存包哪些已经失效了，需要从本地删除。

## 工作流程图


# 现在主要讨论前三个问题: 

## 缓存规则和缓存版本问题 

  缓存配置，我们是用一个json文件来记录的。分本地端配置和服务端配置。 
  * 本地端的配置文件 
  打包APP时获得，或者请求远端接口后保存在本地文件夹中。 resourceList 中没有 diff 字段。主要用于记录本地的离线包文件列表。本地配置文件用 "版本号.json" 文件名存储。
  * 服务器端配置文件 
  在服务器后台配置，前端代码打包后，更新。 
  请求远程配置文件的时候会带上本地配置文件最高版本号字段，服务器端根据版本号判断是否有diff包。打包时可以跟通过参数配置带 diff 包的最低版本，默认跨3个版本，太远的版本必要性也不是很大。 
  如果 diff 包版本字符数过大(这里定位5k)，则放弃 diff 字段，直接全文件下载。 
  * activeTime 
  缓存文件生效时间。可以在版本正式上线前将更新文件缓存到本地。activeTime 时间到的时候资源生效。

  配置文件内容大致如下：  
  
```json
    {
      "version": "4.0.1",
      "activeTime": "版本生效时间，用于预下载升级包",
      "cached": true,
      "resourceList": [
      {
        "downUrl": "资源下载地址",
        "resourceID": "资源id",
        "activeTime": "资源生效时间，用于预下载升级包",
        "type": 3,
        "MIMEType": "text/css",
        "version": 2,
        "matchUrl": "资源匹配正则"
      },
      {
        "downUrl": "资源下载地址",
        "resourceID": "id_mainindexjs",
        "type": 4,
        "MIMEType": "application/x-javascript",
        "version": 4,
        "matchUrl": "资源匹配正则",
        "diff": "补丁包"
      }]
    }
```

  version 用来标识配置的版本号， resourceList 用来存储缓存的资源文件列表。

### request 替换和存储
  * request 拦截 
  IOSAPP 中的网络请求会走苹果的 URL Loading System。 这里用到了 NSURLProtocol 这个协议。具体的使用可以参照网文，要注意的就是:
    * NSURLProtocol 是可以注册多个 NSURLProtocol 子类的，而且是全局的;
    * 还有就是 ** POST请求BODY丢失和协议头跨域 ** 的坑。 
    
  我们的做法是，在 WKWebview 的 decidePolicyForNavigationAction 方法中修改匹配url的协议头为自定义schame。
  为了不影响其他模块的url拦截，最好选取一个比较特别的schame。然后在自定义的 WKWebview 中注册这个 schame： 

```objectivec
  + (void)registerURLProtocol
  {
    if (regURLProtocolCounter == 0){
      [NSURLProtocol registerClass:[DEMOURLProtocol class]];
      [NSURLProtocol wk_registerScheme:@"自定义schame"];
    }
    regURLProtocolCounter ++;
  }
```
  在自定义的 DEMOURLProtocol 的 `- (void)startLoading` 方法中放入自定义的 ResourceLoader。
```objectivec
- (void)startLoading
{
  _startTime = [NSDate date];
  NSMutableURLRequest *mutableReqeust = [[self request] mutableCopy];
  //给我们处理过的请求设置一个标识符, 防止无限循环,
  [NSURLProtocol setProperty:@YES forKey:PLNSURLProtocolHKey inRequest:mutableReqeust];
  NSDictionary* dic = [[PLResourceLoader sharedInstance]responseWithRequest:mutableReqeust];
  if (DICTIONARYHASVALUE(dic)){
    [self.client URLProtocol:self didReceiveResponse:dic[@"response"] cacheStoragePolicy:NSURLCacheStorageAllowed];
    [self.client URLProtocol:self didLoadData:dic[@"data"]];
    [self.client URLProtocolDidFinishLoading:self];
  }else{
    NSURLSession *session = [NSURLSession sessionWithConfiguration:[NSURLSessionConfiguration defaultSessionConfiguration] delegate:self delegateQueue:nil];
    self.task = [session dataTaskWithRequest:self.request];
    [self.task resume];
  }
}
```
关键的方法就是 `- (NSDictionary* )responseWithRequest:(NSURLRequest *)request` ,尝试加载request请求的本地资源，如果没有*可加载的*本地资源，则返回nil。如果有找到*可用的*本地资源，则返回。 
PLResourceLoader 是一个单例类，模块启动的时候会调用 checkUpdate 方法检查更新。
  1. checkUpdate 方法会首先 loadConfig 来加载缓存配置文件到内存中，也就是将上文提到的json文件解析成方便处理的数据结构;
  2. 配置加载完成会校验本地缓存文件，同时请求缓存配置接口。本地文件以 "资源id-版本号" 的格式存储在 tmp 文件夹中。
  3. 接口配置请求到资源后，会将配置文件写到本地，并开启线程在后台下载资源和打diff包。并删除废弃的资源(不在 resourceList 中的文件)。
  4. responseWithRequest 方法中，会根据匹配规则和生效时间寻找本地是否有对应的缓存文件，如果有则生成对应的 response 返回。 

### 跨域问题
H5页面的 URL 换成自定义的scheme后会产生跨域问题，解决方法可以在服务器中 添加自定义的schame来解决。但由于浏览器的安全性问题，非https协议的请求直接无法发出。我在 WKWebview 容器中将js的 `fetch` 方法里将自定义schame换成了 https，因为接口的请求是不需要缓存在本地的。这能能满足大多数接口请求。
```objectivec
  NSDictionary *webParams = @{@"iphonex":@IS_IPHONEX,@"appversion":APP_VERSION,@"os":@"iphone"};
  NSString *javaScriptSource = @"var s_ajaxSwizzle={};s_ajaxSwizzle.tempOpen= XMLHttpRequest.prototype.open;XMLHttpRequest.prototype.open=function(a,b){if(b){b=b.replace(/^(自定义schame\\:\\/\\/)|^(\\/\\/)/,\"https://\")}s_ajaxSwizzle.tempOpen.apply(this,arguments)};";
  javaScriptSource = [javaScriptSource stringByAppendingFormat:@"window.PLParams=%@;",[webParams JSONString]];
  if (IOSVersion_10){
    javaScriptSource = [javaScriptSource stringByAppendingString:@"if(self.fetch){var swizzlefetch=self.fetch;self.fetch=function(a,b){if(a){a=a.replace(/^(自定义schame\\:\\/\\/)|^(\\/\\/)/,\"https://\")} return swizzlefetch.apply(this,arguments)}}"];
  }
  WKUserScript *ajaxSwizzle = [[WKUserScript alloc] initWithSource:javaScriptSource injectionTime:WKUserScriptInjectionTimeAtDocumentStart forMainFrameOnly:YES];
  [webView.configuration.userContentController addUserScript:ajaxSwizzle];
```

## 结尾
这个离线包机制放到线上用的效果还是不错的，web网络加载的时间基本可以忽略了，剩下的基本就是web渲染的时间了。 

文章是把实现IOS代码的主要部分描述出来了，打包相关的和服务端接口的实现都还没出来，以后有时间再写一篇记录下来。 

还差个流程图也没时间画了，以后有空再补上吧。
---
layout: post
title:  "Transmission Between Weex and iOS Native"
date:   2017-03-24 13:00:00 +0800
categories: Tech
summary:    在使用Weex时，我们经常需要数据和事件与native之间的交互，这篇文章描述怎么建立一个统一的交互路由。
---

## Weex 与 iOS Native 的传输

在使用Weex时，我们经常需要数据和事件与native之间的交互，多发生于界面事件和数据传输。举个很简单的例子，Weex page中一项内购的页面，需要显示Apple store的价格，此时需要native从服务器拿到价格后发送给weex page做显示。这个过程包括weex发送请求到本地，本地处理回调等。

本文主要内容就是，如何实现这一过程及保证统一、低耦合的中间层。

### 实现原理

#### Weex 端发送请求
在Weex界面的JS script中，我们能通过调用App注册的模组来直接调用方法。在demo中，处理这一过程的是`XLPluginEvent`类。在初始化SDK时，通过注册：

```objc
[WXSDKEngine registerModule:@"mediator" withClass:NSClassFromString(@"XLPluginEvent")];
```

JS端就可以通过注册的类其中的方法，来发起一个Event：

```js
var module = weex.requireModule('mediator');
module.sendRequestData(data, 'pluginA', 'native', (ret) => {
	this.result = ret
	modal.toast({
		message: 'receive',
		duration: 0.3
	})
})
```
JS端的function可以以参数传递给native并以block形式调用。这样就实现了JS端发起Event并可以获得回调。

#### Native发起Events

Native端可有两种方式传递events至Weex。

-  通过WXSDKManager提供的fireEvent:ref:type:params:domChanges:方法。用这个方法可以在weex page上监听到事件，如
```objc
[[WXSDKManager bridgeMgr] fireEvent:targetWX.InstanceId
                                ref:@"_root" 
                               type:@"nativetransport"
                             params:paramsAddCallback
                         domChanges:nil];
```
{% highlight html %}
  <template>
    <text onClick='onClick' onnativetransport="{{nativeTransport}}" id='text1'>callback: {{result}}</text>
  </template>

  <script>
    # 上略
    nativeTransport: function(e) {
      console.log(e)
    }
  </script>
{% endhighlight %}

-  通过fireGlobalEvent:params:方法
这个方法其实是发送一个全局通知，所有已存在的Weex page都能接收到这个Event，但前提是Weex page初始化时注册listener。
```js
var globalEvent = require('@weex-module/globalEvent')
globalEvent.addEventListener("nativetransport", (e) => {
	console.log(e)
	this.result = "native send a request"
})
```
这时native调用`fireEvent`就可以触发这个监听事件。

### 中间层
建立中间层的目的是统一对所有Weex page的events做转发，一来是方便做hook，二来是防止plugin与plugin之间互相直接调用产生耦合。如果没有统一的中间层和派发关系，想象一下维护过N个版本之后的杂乱无章的调用关系...

梳理下我们期待的流程图：

![image](https://raw.githubusercontent.com/Mioke/resources/master/images/weex_native_transport.png)

#### 绑定事件
每个Weex instance都可以绑定module来处理事件，所以module的实例也是分散形式的。我们用统一的类`XLPluginEvent`来收集信息，并将信息转到统一的中间层类`XLWxPluginMediator`做处理。

```objc
[WXSDKEngine registerModule:@"mediator" withClass:NSClassFromString(@"XLPluginEvent")];

// In XLPluginEvent class:
- (void)sendRequestData:(NSDictionary *)data
                   from:(NSString *)source
               toTarget:(NSString *)target
               callback:(WXModuleCallback)callback {
    [[XLWxPluginMediator sharedInstance] sendRequestData:data
                                                    from:weexInstance
                                                  source:source
                                                toTarget:target
                                                callback:callback];
}

```

#### 中间处理
`XLWxPluginMediator`同时具有接收和发送功能。接收event之后会根据event附带的information，做本地分发或插件分发，一般用`target`标明event的目标对象。
```objc
- (void)sendRequestData:(NSDictionary *)data
                   from:(WXSDKInstance *)wx
                 source:(NSString *)source
               toTarget:(NSString *)target
               callback:(WXModuleCallback)callback {
    if ([target isEqualToString:@"native"]) {
        // Notify all native listeners
        
    } else {
        // 1 . find target
//        WXSDKInstance *target = [Finder findtarget];
        // 2 . fire a event & wait to callback
//        [self sendEventTo:target params:data callback:callback];
    }
}

```
native向plugin做通信请求**暂时**是通过fireGlobalEvent方法实现的，原因是使用fireEvent会与**界面绑定**，某些场合我们并不需要与界面绑定。

```objc
- (void)sendEventTo:(WXSDKInstance *)wx 
             params:(NSDictionary *)params
           callback:(WXModuleCallback)callback {
           
    NSMutableDictionary *paramsAddCallback = [NSMutableDictionary dictionaryWithDictionary:params];
    [paramsAddCallback setObject:callback forKey:@"callback"];
    
    [wx fireGlobalEvent:@"nativetransport" params:paramsAddCallback];
}
```
拥有响应native event的plugin必须在onCreate里实现这个event的监听：
```js
created: function () {
            var globalEvent = require('@weex-module/globalEvent')
            globalEvent.addEventListener("nativetransport", (e) => {
                console.log(e)
            })
        },
```

由此就实现了plugin与native之间的双向传输。

#### Plugins 之间的传输

```objc
// TODO: - 
```
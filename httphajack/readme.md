
# 移动端防劫持方案

> 社招移动端还未上线，开始受到运营商wap 劫持，H5页面被插入各种广告，以及弹窗。有下面几种常见的：

![](https://raw.githubusercontent.com/410675629/webpackBundle/master/httphajack/img/1.png)

还有这样的，\

![](https://raw.githubusercontent.com/410675629/webpackBundle/master/httphajack/img/2.png)

> 第一种，还可以忍受，第二种简直不能忍，

> 不能忍，那要咋弄！于是得找人背锅吧

+ 最初的想法，投诉运营商。 不行!!!

+ 最好的方案就是 讲网络传输协议改为 **https 安全协议**，一劳永逸。

<p> 被告知，需要一周时间走流程，真是当头一棒，还只能忍着，可是马上临近要发布上线了，咋搞？看来只能指望自己了。</p>

> 习总说了：**“大家撸起袖子加油干！”** 

***

于是乎，开始上网查资料。总得来说 就是从服务端返回的页面被篡改并插入广告。篡改的方式，大致有以下几种,

1、在页面底部插入script标签引入广告js\
2、直接插入广告内容到页面\
3、劫持正常js请求，返回广告内容js\

所以我们的策略是如何阻止这些带有广告的js 以及图片的加载

### 内容安全策略 (CSP) 

实质是白名单制度。开发者明确告诉客户端，哪些外部资源可以加载和执行，等同于提供白名单。它的实现和执行全部由浏览器完成，开发者只需提供配置。

有两种方案可以启用CSP
> 一种是通过 HTTP 头信息的Content-Security-Policy的字段

![](https://raw.githubusercontent.com/410675629/webpackBundle/master/httphajack/img/3.png)

> 另一种 操作比较简单：
```
<meta http-equiv="Content-Security-Policy" content="script-src 'self'; object-src 'none'; style-src cdn.example.org third-party.org; child-src https:">
```

**配置的目的：**

 +   脚本：只信任当前域名
 +  <object>标签：不信任任何URL，即不加载任何资源
 +   样式表：只信任cdn.example.org和third-party.org
 +   框架（frame）：必须使用HTTPS协议加载
 +  其他资源：没有限制

项目一旦启动：
```<script src="http://res.wx.qq.com/open/js/jweixin-1.0.0.js"></script>```  报错。
<p>下载下来，改成当前域能解决问题吗？,所以引用外部资源cdn需要全部改为本地的。（先暂且不说这个好不好...）</p>

<p>由于是移动web,之前有个需求，就是要统计用户的数据，所以里面有大量使用动态脚本注入，如下所示：</p>
```
<script>
    window.NRUM = window.NRUM || {};
    window.NRUM.config = {key:'67ed58f43b424977b0fd742c0002ff88',clientStart: +new Date()};
    (function() {var n = document.getElementsByTagName('script')[0],s = document.createElement('script');s.type = 'text/javascript';s.async = true;s.src = '//nos.netease.com/apmsdk/napm-web-min-1.1.6.js';n.parentNode.insertBefore(s, n);})();
</script>
```
这种情况，csp处理不来，直接给过滤了

> CSP-api 能限制的有以下选项。
```
script-src：外部脚本
style-src：样式表
img-src：图像
media-src：媒体文件（音频和视频）
font-src：字体文件
object-src：插件（比如 Flash）
child-src：框架
frame-ancestors：嵌入的外部资源（比如<frame>、<iframe>、<embed>和<applet>）
connect-src：HTTP 连接（通过 XHR、WebSockets、EventSource等）
worker-src：worker脚本
manifest-src：manifest 文件

```
***
哪咋办呢？那就写代码解决。

```
防范范围：
   1）所有内联事件执行的代码
   2）href 属性 javascript: 内嵌的代码
   3）静态脚本文件内容
   4）动态添加的脚本文件内容
   5）document-write添加的内容
   6）iframe嵌套
```
```
1、设置安全域名
var safeList =[/netease|app|hr.163|vendor|manifest/g]

2 过滤class关键词
    var filterClassName = [
        'BAIDU_DUP_wrapper', //百度推广
        'BAIDU_DSPUI_FLOWBAR'
];

 3、 过滤name关键词
    var filterProName = [
        'text',
        '#text',
        'IFRAME',
        'SCRIPT',
        'IMG'
    ];
```
### 主要分为以下5步骤
```
    [1] scanInlineElement && inlineEventFilter();
    [2] interceptionStaticScript();
    [3] lockCallAndApply();
   ``` 
 + 详细介绍
 ```
    //scanInlineElement : false 是否需要扫描内联事件 默认不需要
    判断： 是否内联事件劫持
    function inlineEventFilter() {
        var i = 0,
            obj = null;

        for (obj in document) {
            if (/^on./.test(obj)) {
                interceptionInlineEvent(obj, i++);
            }
        }
    }
    
    /**
     * 内联事件拦截
     * @param  {[String]} eventName [内联事件名]
     * @param  {[Number]} eventID   [内联事件id]
     * @return {[type]}             [description]
     */
    function interceptionInlineEvent(eventName, eventID) {
        var isClick = (eventName === 'onclick');

        document.addEventListener(eventName.substr(2), function(e) {
            /**
             * 扫描元素是否存在内联事件
             * @param  {[DOM]} elem [DOM元素]
             * @param  {[Boolean]} isClick [是否是内联点击事件]
             * @param  {[String]} eventName [内联 on* 事件名]
             * @param  {[Number]} eventID [给每个内联 on* 事件一个id]
            /
            scanElement(e.target, isClick, eventName, eventID);
            
            // 完成的功能 ：扫描包括 a iframe img video div 等所有可以写内联事件的元素
            // [1] 注销事件 elem[eventName] = null;
            // [2] 注销代码 elem.href = 'javascript:void(0)';
            
         }, true);
    }
```   

+ 主动防御

```
/**
     * 主动防御 MutationEvent
     * 使用 MutationObserver 进行静态插入脚本的拦截
     * @return {[type]} [description]
     */
    function interceptionStaticScript() {
        var MutationObserver = window.MutationObserver || window.WebKitMutationObserver || window.MozMutationObserver;
        // 该构造函数用来实例化一个新的 Mutation 观察者对象 Mutation 观察者对象能监听在某个范围内的 DOM 树变化
        if (!MutationObserver) return;
        var observer = new MutationObserver(function(mutations) {
            mutations.forEach(function(mutation) {
                var nodes = mutation.addedNodes;

                // 逐个遍历
                for (var i = 0; i < nodes.length; i++) {
                    var node = nodes[i];
                    // 扫描 script 与 iframe
                    if (node.tagName === 'SCRIPT' || node.tagName === 'IFRAME') {
                        // 拦截到可疑iframe
                        if (node.tagName === 'IFRAME' && node.src && !filter(safeList, node.src)) {
                            node.parentNode && node.parentNode.removeChild(node);
                            // hiidoStat('', 'insertIFRMAETag', '', node.src); //上报资源
                           // console.log('拦截到可疑iframe', node.src);

                        } else if (node.src) {
                            // 只放行白名单
                            if (!filter(safeList, node.src)) {
                                node.parentNode && node.parentNode.removeChild(node);
                                hiidoStat(node.src, 'insertScriptTag', '', '');
                                // console.log('拦截可疑静态脚本:', node.src);
                            }
                        }
                    }
                }
            });
        });
        // 传入目标节点和观察选项
        // 如果 target 为 document 或者 document.documentElement
        // 则当前文档中所有的节点添加与删除操作都会被观察到d
        observer.observe(document, {
            subtree: true,
            childList: true
        });
    }
```
+ ```Object.defineProperty```，锁住```call```和```apply```，使之无法被重写
```
   /**
     * 使用 Object.defineProperty，锁住call和apply，使之无法被重写
     * @return {[type]} [description]
     */
    function lockCallAndApply() {
        // 锁住 call
        try {
            Object.defineProperty(Function.prototype, 'call', {
                value: Function.prototype.call,
                // 当且仅当仅当该属性的 writable 为 true 时，该属性才能被赋值运算符改变
                writable: false,
                // 当且仅当该属性的 configurable 为 true 时，该属性才能够被改变，也能够被删除
                configurable: false,
                enumerable: true
            });
            // 锁住 apply
            Object.defineProperty(Function.prototype, 'apply', {
                value: Function.prototype.apply,
                writable: false,
                configurable: false,
                enumerable: true
            });
        } catch (e) {
            // console && console.log(e);
        }

    }
```

> 性能统计上报
```
    /**
     * 统计上报函数
     * @param  {[type]} url 拦截脚本地址
     * @param  {[type]} className 拦截插入元素className
     * @param  {[type]} eName 内联事件名称
     * @param  {[type]} fUrl ifrmae乔套url
     */
(new Image).src = 'http://m.hr.163.com/report?' + queryStr
```

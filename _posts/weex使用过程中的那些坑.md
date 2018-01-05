

####前言：公司采用weex来开发项目骑手工程已经有3个多月了，目前在点我达骑手工程已有超过%70的页面向weex过度完成，期间也遇到了很多weex本身的一些坑，借此机会来记录下的这个虐心的过程，希望能帮助到正在使用weex开发的小伙伴们少走些弯路。

######问题1：weex的stream模块的fetch方法在弱网环境下容易请求失败？
- 场景：点我达骑手工程在11月中旬之后线上weex页面大量覆盖，在之后的5个城市进行灰发的一周里雷霆群大量骑手反馈android和iOS骑手客户端的weex页面出现大面积的网络请求失败的状况。

- 紧急预案：在接到骑手反馈之后，我们启动紧急兜底方案，下架所有灰发城市的weex页面，并降级为native页面。

- 问题收集：由于之前并没考虑到在weex里预埋对接口相关的异常日志信息的收集以及统计，只能依靠kibana来查找相关的骑手的请求，经过一番查询并没有在异常的时间节点发现有上报的weex请求，只能说明出现问题的骑手的weex请求根本没有发出来，导致这个问题无法追踪。无奈之下我们决定在选择一个流量比较大weex页面在杭州和深圳这两个灰发城市作为试点，并在这个weex页面植入网络请求失败的统计收集信息包括status(weex网络请求的状态码),statusText,timestamp(时间戳)，api(请求失败的接口名称)等,在网络好的时候上报数据，通过这个方式来采集这部分异常数据。

1.收集的数据的代码
```
function get(url) {
    return new Promise((resolve, reject) => {
        stream.fetch({
            method: 'GET',
            url: url,
            type: 'json'
        }, (response) => {
            if (response.status == 200) {
                // resolve(response.data)
                if (response.data.status==1) {
                    if(response.data.data){
                        resolve(response.data.data)
                    }else{
                        resolve({})
                    }
                } else {
                    reject(response.data)
                }
            }
            else {
                var timestamp = Date.parse(new Date());
                timestamp = timestamp / 1000;
                var api = url.split('?')[0].split('/')
                var msg = {
                    msg : '网络异常，请稍后重试',
                    data:{
                        httpStatus:''+response.status,
                        statusText:response.statusText,
                        headers:response.headers,
                        data:response.data,
                        url:api[api.length-1],
                        timestamp:''+timestamp
                    }

                }
                reject(msg)
            }
        }, () => {})
    })
}
```

```
            httpFaildLogUpload() {
                storage.getItem(config.httpFaildMsg, event => {
                    if(event.data!='undefined') {
                        var msg = ''
                        if (this.instance.isAndroid()) {
                            msg  = event.data
                        } else if (this.instance.isIOS()) {
                            let Base64 = require('js-base64').Base64;
                            msg = Base64.encode(event.data)
                        }
                        // Base64.encode(event.data)
                        this.$store.dispatch('monitorAction',{'weexhttpfaildLog':msg}).then(successData =>{                            storage.removeItem(config.httpFaildMsg)
                        }, failData=>{
                            if(failData.msg){
                      this.common.toast(failData.msg)
                            }
                        })
                    }
                })
            }
```
2.收集的数据格式如下
```
{
    "httpStatus": "-1",
    "statusText": "",
    "data": "似乎已断开与互联网的连接。(-1009)",
    "url": "get-rider-info.json",
    "timestamp": "1512001928"
}
```


- 问题分析：发现收集上来的数据status的状态值都是-1的情况，并且weexSDK返回的提示信息为“似乎已断开与互联网的连接。(-1009)”，面对这种情况。
 - 猜想1：是不是真的是网络断开了呢？为了能解答这个问题，我们拿着weex请求失败时记录的时间戳对比了native在这个时间戳范围内的网络请求，然而数据显示native的网络请求是正常的。

 - 猜想2：是不是由于weex的链路比较长导致的呢？但是马上否定了这一猜想，链路的长短只是影响应用发出网络请求时间的快慢，并不影响请求本身，更何况是请求根本没有抵达服务端就失败了。

 - 猜想3：是不是weex请求在弱网络环境下容易请求超时？那么weex的请求超时时间是多少呢？

带着这一猜想开始模拟我的猜想

1.使用charles模拟弱网环境
![alt](http://tech.dianwoda.com/content/images/2017/12/----.png)
发现在15kb以下的网络下weex有%80的网络超时的概率，而native只有%20的网络超时概率

2.查看weex官方文档对stream的fetch方法的说明，并没有发现特殊的说明。
![alt](http://tech.dianwoda.com/content/images/2017/12/-----1.png)
3.阅读weex的stream模块的android和iOS源码

iOS端源码分析

iOS中抛给weex调用的方法
```
- (void)fetch:(NSDictionary *)options callback:(WXModuleCallback)callback progressCallback:(WXModuleKeepAliveCallback)progressCallback
{
    __block NSInteger received = 0;
    __block NSHTTPURLResponse *httpResponse = nil;
    __block NSMutableDictionary * callbackRsp =[[NSMutableDictionary alloc] init];
    __block NSString *statusText = @"ERR_CONNECT_FAILED";
    
    //build stream request，生成request
    WXResourceRequest * request = [self _buildRequestWithOptions:options callbackRsp:callbackRsp];
    if (!request) {
        if (callback) {
            callback(callbackRsp);
        }
        // failed with some invaild inputs
        return ;
    }
    
    // notify to start request state
    if (progressCallback) {
        progressCallback(callbackRsp, TRUE);
    }
    // 挂载请求
    WXResourceLoader *loader = [[WXResourceLoader alloc] initWithRequest:request];
    __weak typeof(self) weakSelf = self;
    // 接收到相应
    loader.onResponseReceived = ^(const WXResourceResponse *response) {
        httpResponse = (NSHTTPURLResponse*)response;
        if (weakSelf) {
            [callbackRsp setObject:@{ @"HEADERS_RECEIVED" : @2 } forKey:@"readyState"];
            [callbackRsp setObject:[NSNumber numberWithInteger:httpResponse.statusCode] forKey:@"status"];
            [callbackRsp setObject:httpResponse.allHeaderFields forKey:@"headers"];
            statusText = [WXStreamModule _getStatusText:httpResponse.statusCode];
            [callbackRsp setObject:statusText forKey:@"statusText"];
            [callbackRsp setObject:[NSNumber numberWithInteger:received] forKey:@"length"];
            if (progressCallback) {
                progressCallback(callbackRsp, TRUE);
            }
        }
    };
    // 对接收到的数据进行处理
    loader.onDataReceived = ^(NSData *data) {
        if (weakSelf) {
            [callbackRsp setObject:@{ @"LOADING" : @3 } forKey:@"readyState"];
            received += [data length];
            [callbackRsp setObject:[NSNumber numberWithInteger:received] forKey:@"length"];
            if (progressCallback) {
                progressCallback(callbackRsp, TRUE);
            }
        }
    };
    // 完成请求
    loader.onFinished = ^(const WXResourceResponse * response, NSData *data) {
        if (weakSelf) {
            [weakSelf _loadFinishWithResponse:[response copy] data:data callbackRsp:callbackRsp];
            if (callback) {
                callback(callbackRsp);
            }
        }
    };
    // 请求失败
    loader.onFailed = ^(NSError *error) {
        if (weakSelf) {
            [weakSelf _loadFailedWithError:error callbackRsp:callbackRsp];
            if (callback) {
                callback(callbackRsp);
            }
        }
    };
    
    [loader start];
}

```
weex请求生成，发现需要传递timeout超时时间，而文档并没有提及，那么默认的超时时间是多少呢？
```

- (WXResourceRequest*)_buildRequestWithOptions:(NSDictionary*)options callbackRsp:(NSMutableDictionary*)callbackRsp
{
    // parse request url
    NSString *urlStr = [options objectForKey:@"url"];
    NSMutableString *newUrlStr = [urlStr mutableCopy];
    WX_REWRITE_URL(urlStr, WXResourceTypeLink, self.weexInstance, &newUrlStr)
    urlStr = newUrlStr;
    
    if (!options || [WXUtility isBlankString:urlStr]) {
        [callbackRsp setObject:@(-1) forKey:@"status"];
        [callbackRsp setObject:@NO forKey:@"ok"];
        
        return nil;
    }
    // 生成request
    WXResourceRequest *request = [WXResourceRequest requestWithURL:[NSURL URLWithString:urlStr] resourceType:WXResourceTypeOthers referrer:nil cachePolicy:NSURLRequestUseProtocolCachePolicy];
    
    // parse http method，post或get请求的
    NSString *method = [options objectForKey:@"method"];
    if ([WXUtility isBlankString:method]) {
        // default HTTP method is GET
        method = @"GET";
    }
    request.HTTPMethod = method;
    
    //parse responseType
    NSString *responseType = [options objectForKey:@"type"];
    if ([responseType isKindOfClass:[NSString class]]) {
        [callbackRsp setObject:responseType? responseType.lowercaseString:@"" forKey:@"responseType"];
    }
    
    //parse timeout，可以设置超时时间
    if ([options valueForKey:@"timeout"]){
        //the time unit is ms
        [request setTimeoutInterval:([[options valueForKey:@"timeout"] floatValue])/1000];
    }
    
    //install client userAgent
    request.userAgent = [WXUtility userAgent];
    
    // parse custom http headers
    NSDictionary *headers = [options objectForKey:@"headers"];
    for (NSString *header in headers) {
        NSString *value = [headers objectForKey:header];
        [request setValue:value forHTTPHeaderField:header];
    }
    
    //parse custom body
    if ([options objectForKey:@"body"]) {
        NSData * body = nil;
        if ([[options objectForKey:@"body"] isKindOfClass:[NSString class]]) {
            // compatible with the string body
            body = [[options objectForKey:@"body"] dataUsingEncoding:NSUTF8StringEncoding];
        }
        if ([[options objectForKey:@"body"] isKindOfClass:[NSDictionary class]]) {
            body = [[WXUtility JSONString:[options objectForKey:@"body"]] dataUsingEncoding:NSUTF8StringEncoding];
        }
        if (!body) {
            [callbackRsp setObject:@(-1) forKey:@"status"];
            [callbackRsp setObject:@NO forKey:@"ok"];
            return nil;
        }
        
        [request setHTTPBody:body];
    }
    
    [callbackRsp setObject:@{ @"OPENED": @1 } forKey:@"readyState"];
    
    return request;
}
```
继续看weex默认设置的请求超时时间是多少？发现NSURLSession的NSURLSessionConfiguration采用的是defaultSessionConfiguration的配置，并没有设置
![alt](http://tech.dianwoda.com/content/images/2017/12/-----2.png)
```

- (void)sendRequest:(WXResourceRequest *)request withDelegate:(id<WXResourceRequestDelegate>)delegate
{
    if (!_session) {
        // NSURLSession默认的配置
        NSURLSessionConfiguration *urlSessionConfig = [NSURLSessionConfiguration defaultSessionConfiguration];
        if ([WXAppConfiguration customizeProtocolClasses].count > 0) {
            NSArray *defaultProtocols = urlSessionConfig.protocolClasses;
            urlSessionConfig.protocolClasses = [[WXAppConfiguration customizeProtocolClasses] arrayByAddingObjectsFromArray:defaultProtocols];
        }

        _session = [NSURLSession sessionWithConfiguration:urlSessionConfig
                                                 delegate:self
                                            delegateQueue:[NSOperationQueue mainQueue]];
        _delegates = [WXThreadSafeMutableDictionary new];
    }
    
    NSURLSessionDataTask *task = [_session dataTaskWithRequest:request];
    request.taskIdentifier = task;
    [_delegates setObject:delegate forKey:task];
    [task resume];
}
```
根据以上iOS的源码分析，当我们没有在weex传入timeout这个参数时一下iOS中的超时时间解释，可以看出

```
    @abstract Sets the timeout interval of the receiver.
    @discussion The timeout interval specifies the limit on the idle
    interval allotted to a request in the process of loading. The "idle
    interval" is defined as the period of time that has passed since the
    last instance of load activity occurred for a request that is in the
    process of loading. Hence, when an instance of load activity occurs
    (e.g. bytes are received from the network for a request), the idle
    interval for a request is reset to 0. If the idle interval ever
    becomes greater than or equal to the timeout interval, the request
    is considered to have timed out. This timeout interval is measured
    in seconds.
翻译：
@设置接收器的超时间隔。@讨论超时间隔，指定空闲时间的限制。在加载过程中分配给请求的间隔。“闲置“间隔”定义为自发生在一个请求中的负载活动的最后实例。加载过程。因此，当发生负载活动实例时（例如，从网络接收字节请求）、空闲请求的间隔被重置为0。如果空闲时间间隔大于或等于超时间隔，请求被认为是超时的。此超时间隔是测量的。在几秒钟内。
```
android中的源码分析：

android中抛给weex调用的方法
```
  /**
   *
   * @param optionsStr request options include:
   *  method: GET 、POST、PUT、DELETE、HEAD、PATCH
   *  headers：object，请求header
   *  url:
   *  body: "Any body that you want to add to your request"
   *  type: json、text、jsonp（native实现时等价与json）
   * @param callback finished callback,response object:
   *  status：status code
   *  ok：boolean 是否成功，等价于status200～299
   *  statusText：状态消息，用于定位具体错误原因
   *  data: 响应数据，当请求option中type为json，时data为object，否则data为string类型
   *  headers: object 响应头
   *
   * @param progressCallback in progress callback,for download progress and request state,response object:
   *  readyState: number 请求状态，1 OPENED，开始连接；2 HEADERS_RECEIVED；3 LOADING
   *  status：status code
   *  length：当前获取的字节数，总长度从headers里「Content-Length」获取
   *  statusText：状态消息，用于定位具体错误原因
   *  headers: object 响应头
   */
  @JSMethod(uiThread = false)
  public void fetch(String optionsStr, final JSCallback callback, JSCallback progressCallback){

    JSONObject optionsObj = null;
    try {
      optionsObj = JSON.parseObject(optionsStr);
    }catch (JSONException e){
      WXLogUtils.e("", e);
    }

    boolean invaildOption = optionsObj==null || optionsObj.getString("url")==null;
    if(invaildOption){
      if(callback != null) {
        Map<String, Object> resp = new HashMap<>();
        resp.put("ok", false);
        resp.put(STATUS_TEXT, Status.ERR_INVALID_REQUEST);
        callback.invoke(resp);
      }
      return;
    }
    String method = optionsObj.getString("method");
    String url = optionsObj.getString("url");
    JSONObject headers = optionsObj.getJSONObject("headers");
    String body = optionsObj.getString("body");
    String type = optionsObj.getString("type");
    int timeout = optionsObj.getIntValue("timeout");

    if (method != null) method = method.toUpperCase();
    Options.Builder builder = new Options.Builder()
            .setMethod(!"GET".equals(method)
                    &&!"POST".equals(method)
                    &&!"PUT".equals(method)
                    &&!"DELETE".equals(method)
                    &&!"HEAD".equals(method)
                    &&!"PATCH".equals(method)?"GET":method)
            .setUrl(url)
            .setBody(body)
            .setType(type)
            .setTimeout(timeout);

    extractHeaders(headers,builder);
    final Options options = builder.createOptions();
    sendRequest(options, new ResponseCallback() {
      @Override
      public void onResponse(WXResponse response, Map<String, String> headers) {
        if(callback != null) {
          Map<String, Object> resp = new HashMap<>();
          if(response == null|| "-1".equals(response.statusCode)){
            resp.put(STATUS,-1);
            resp.put(STATUS_TEXT,Status.ERR_CONNECT_FAILED);
          }else {
            int code = Integer.parseInt(response.statusCode);
            resp.put(STATUS, code);
            resp.put("ok", (code >= 200 && code <= 299));
            if (response.originalData == null) {
              resp.put("data", null);
            } else {
              String respData = readAsString(response.originalData,
                      headers != null ? getHeader(headers, "Content-Type") : ""
              );
              try {
                resp.put("data", parseData(respData, options.getType()));
              } catch (JSONException exception) {
                WXLogUtils.e("", exception);
                resp.put("ok", false);
                resp.put("data","{'err':'Data parse failed!'}");
              }
            }
            resp.put(STATUS_TEXT, Status.getStatusText(response.statusCode));
          }
          resp.put("headers", headers);
          callback.invoke(resp);
        }
      }
    }, progressCallback);
  }
```


查看默认的请求超时时间
```
  private Options(String method,
                  String url,
                  Map<String, String> headers,
                  String body,
                  Type type,
                  int timeout) {
    this.method = method;
    this.url = url;
    this.headers = headers;
    this.body = body;
    this.type = type;
    if (timeout == 0) {
      timeout = WXRequest.DEFAULT_TIMEOUT_MS;
    }
    this.timeout = timeout;
  }
```
android默认的请求超时时间为3秒
```
  private Options(String method,
                  String url,
                  Map<String, String> headers,
                  String body,
                  Type type,
                  int timeout) {
    this.method = method;
    this.url = url;
    this.headers = headers;
    this.body = body;
    this.type = type;
    if (timeout == 0) {
      timeout = WXRequest.DEFAULT_TIMEOUT_MS;
    }
    this.timeout = timeout;
  }
```


以上分析iOS和android默认不设置超时时间，那么请求的超时时间只有几秒钟，所有在弱网环境下比较容易造成请求超时的情况。

- 问题解决：在stream.fetch()传入timeout请求超时时间为20秒，问题完美解决。

######问题2：weex的picker模块在android端存在严重crash和不相应的严重问题
- 场景：在使用时间选择的界面中
- 问题：经测试发现

 - pick方法在android和iOS都可以正常使用
 - pickDate只支持iOS，在android直接crash
 - pickTime只支持iOS，在android直接不响应

- 解决方案：如果要使用pickDate，pickTime时间选择方法只能自己定义

######问题3：weex的list控件在少量数据条数下同时上拉和下拉android直接卡住不能滑动，iOS直接悬挂在中间。
- 场景：在列表界面中
解决方案：
iOS
1.使用hook WXScrollerComponent的new_initWithRef，updateAttributes，viewDidLoad，替换refresh和loading控件，并使用fireEvent来抛出刷事件

步骤：

1.hook WXScrollerComponent的new_initWithRef，updateAttributes，viewDidLoad
```
+ (void)load {
    [self weex_swizzle:[self class] Method:@selector(initWithRef:type:styles:attributes:events:weexInstance:) withMethod:@selector(new_initWithRef:type:styles:attributes:events:weexInstance:)];
    [self weex_swizzle:[self class] Method:@selector(updateAttributes:) withMethod:@selector(new_updateAttributes:)];
    [self weex_swizzle:[self class] Method:@selector(viewDidLoad) withMethod:@selector(new_viewDidLoad)];
}

-(instancetype)new_initWithRef:(NSString *)ref type:(NSString *)type styles:(NSDictionary *)styles attributes:(NSDictionary *)attributes events:(NSArray *)events weexInstance:(WXSDKInstance *)weexInstance {
    id ret = [self new_initWithRef:ref type:type styles:styles attributes:attributes events:events weexInstance:weexInstance];
    for (NSString *e in events) {
        if ([e isEqualToString:@"refresh"]) {
            self.headerExist = YES;
        }
        if ([e isEqualToString:@"loading"]) {
            self.footerExist = YES;
        }
    }
    return ret;
}

- (void)new_updateAttributes:(NSDictionary *)attributes {
    [self new_updateAttributes:attributes];
    
    if (attributes[@"refreshDisplay"]) {
        if ([attributes[@"refreshDisplay"] isEqualToString:@"hide"]) {
            [(UIScrollView *)self.view headerStopRefresh];
            [UIView animateWithDuration:0.4 animations:^{
                [(UIScrollView *)self.view setContentOffset:CGPointZero];
            }];
        }
    }
    if (attributes[@"loadingDisplay"]) {
        if ([attributes[@"loadingDisplay"] isEqualToString:@"hide"]) {
            [(UIScrollView *)self.view footerStopRefresh];
        }
    }
}

- (void)new_viewDidLoad {
    [self new_viewDidLoad];
    
    if (self.headerExist) {
        MJRefreshNormalHeader *header = [MJRefreshNormalHeader headerWithRefreshingTarget:self refreshingAction:@selector(refresh)];
        [(UIScrollView *)self.view setMj_header:header];
    }
    
    if (self.footerExist) {
        MJRefreshAutoNormalFooter *footer = [MJRefreshAutoNormalFooter footerWithRefreshingTarget:self refreshingAction:@selector(loading)];
        [(UIScrollView *)self.view setMj_footer:footer];
    }
}

```

2.fireEvent来抛出刷事件给weex
```
- (void)refresh {
    [self fireEvent:@"refresh" params:nil];
}

- (void)loading {
    [self fireEvent:@"loading" params:nil];
}
```

android
解决方案：android中的问题是由于android上拉刷新和下拉刷新的判断由同一个字段来维护的，当上拉刷新和下拉刷新同时出现时就会卡主无法识别，需要修改源码，已经反馈给weex官方。


######问题4：weex使用v-if语法时需要注意在v-if里面的判断内容不加上(),在android部分机型中无法识别
解决方案：使用v-if中加上()

######问题5：weex使用css样式不要这样命名如div，div1，div2，这样做在切换样式时会出现样式混乱的情况
解决方案：不使用1，2，3，这样顺序命名

######问题6：weex使用div里面的内容超出控件部分在android端会被截掉。
如需要完成这样的页面：
![alt](http://tech.dianwoda.com/content/images/2017/12/-----4.png)
解决方案:可以使用padding来外扩区域


######结语：以上是笔者总结的一部分坑，当然还有很多，由于公司比较忙没有这么多时间全部写完，有空闲我会慢慢把其他的一些遇到的坑补充上去，希望能帮助到正在使用weex开发的小伙伴们少走些弯路。

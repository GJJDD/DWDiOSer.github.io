**前言：公司采用weex对骑手工程改造已经有3个多月了，随着骑手工程中weex页面比例不断增大，对weex来实现复杂UI的需求也越来越强烈，但是目前weex对绘图功能一片空白，插件市场的插件长期没人维护，开发者苦不堪言，我们对此进行了探索，出于对android，iOS，web兼容的考虑，我们选择了svg技术对weex进行插件扩展的研究，借此机会来记录下的这个虐心的过程，希望能帮助到正在使用weex开发的小伙伴们少走些弯路。**
######目录
* **1.SVG简介**
 * 1.1 什么是SVG？
 * 1.2 SVG工作流？
 * 1.3 为什么选择SVG？
 * 1.4 SVG优势
 * 1.5 SVG实例
 * 1.6 SVG预定义的形状元素
* **2. Weex 插件扩展（有相关经验的小伙伴可忽略）**
 * 2.1 weex-iOS插件扩展方式介绍
 * 2.2 weex-android插件扩展方式介绍
 * 2.3 weex-web插件扩展方式介绍
* **3.Weex 交互（有相关经验的小伙伴可忽略）**
  * 3.1 Weex-web和native交互
  * 3.2 Weex和iOS交互
  * 3.3 Weex和android交互
* **4.Weex SVG实践**
 * 4.1 Weex-web SVG扩展
 * 4.2 Weex-iOS SVG扩展
 * 4.3 Weex-android SVG扩展
* **5.SVG编辑工具介绍**
 * 5.1 SVGSUS
* **6.Weex 开发过程中遇到的一些坑**
######1.SVG简介
![alt](http://tech.dianwoda.com/content/images/2018/01/u-1889500682-3133816136-fm-27-gp-0.jpg)

* **1.1 什么是SVG？**
 * SVG指可伸缩矢量图形（Scalable Vector Graphics）
 * SVG用来定义用于网络的基于矢量的图形；
 * SVG使用xml格式定义图形；
 * SVG图像在放大或改变尺寸的情况下其图形质量不会有所损失；
 * SVG是万维网联盟的标准；
 * SVG与诸如DOM和XSL之类的W3C标准是一个整体（于2003年1月14日成为W3C推荐标准）。
* **1.2 SVG工作流？**
![alt](http://tech.dianwoda.com/content/images/2018/01/4e4a20a4462309f7fe64561a720e0cf3d7cad6ab---1.png)
* **1.3 为什么选择SVG？**
 *  SVG是基于XML与HTML语义一样，具有很好的交互性，图像文件可读，易于修改和编辑。
 * 目前主流的平台及android，iOS，web都有比较好的支持。
* **1.4 SVG优势**
 *  可通过文本编辑器来创建和修改
 *  图像中的文本是可选的，同时也是可搜索的（很适合制作地图）
 *  SVG可在任何的分辨率下被高质量的打印
 *  可在图片质量不下降的情况下被放大
 *  SVG 与 JPEG 和 GIF 图像比起来，尺寸更小，且可压缩性更强。
 *  支持事件绑定
 *  SVG 可以与 Java 技术一起运行
 *  SVG 是开放的标准
 *  SVG 文件是纯粹的 XML
* **1.5 SVG实例**
```
<?xml version="1.0" standalone="no"?>
//第一行包含了XML声明。standalone 该属性规定此SVG文件是否是“独立“，或含有对外部文件的引用。
//standalone="no" 意味着 SVG 文档会引用一个外部文件 - 在这里，是 DTD 文件。

<!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" 
"http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">

//第二和第三行引用了这个外部的 SVG DTD。该 DTD 位于 //“http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd”。
//该 DTD 位于 W3C，含有所有允许的 SVG 元素。

<svg width="100%" height="100%" version="1.1"
xmlns="http://www.w3.org/2000/svg">

//SVG 代码以 <svg> 元素开始，包括开启标签 <svg> 和关闭标签 </svg> 这是根元素。width 和 height 属
//性可设置此 SVG 文档的宽度和高度。version 属性可定义所使用的 SVG 版本，xmlns 属性可定义 SVG 命名空间。

<circle cx="100" cy="50" r="40" stroke="black" stroke-width="2" fill="red"/>

//SVG 的 <circle> 用来创建一个圆。cx 和 cy 属性定义圆中心的 x 和 y 坐标。如果忽略这两个属性，
//那么圆点会被设置为 (0, 0)。r 属性定义圆的半径。
//stroke 和 stroke-width 属性控制如何显示形状的轮廓。我们把圆的轮廓设置为 2px 宽，黑边框。
//fill 属性设置形状内的颜色。我们把填充颜色设置为红色。
//关闭标签的作用是关闭 SVG 元素和文档本身。

</svg>
//注释：所有的开启标签必须有关闭标签！
```
* **1.6 SVG预定义的形状元素**
 * 矩形 `<rect>`
 * 圆形 `<circle>`
 * 椭圆 `<ellipse>`
 * 线 `<line>`
 * 折线 `<polyline>`
 * 多边形 `<polygon>`
 * 路径 `<path>`
######2. Weex 插件扩展（有相关经验的小伙伴可忽略）
![alt](http://tech.dianwoda.com/content/images/2018/01/u-1942997262-987558119-fm-11-gp-0-jpg.png)
 * **2.1 weex-iOS插件扩展方式介绍**

iOS组件`Component`扩展步骤
```
iOS端
*1.创建自定义的组件继承WXComponent类
*2.重写- (instancetype)initWithRef:(NSString *)ref type:(NSString *)type styles:(NSDictionary *)styles attributes:(NSDictionary *)attributes events:(NSArray *)events weexInstance:(WXSDKInstance *)weexInstance方法
*3.现实自定义的UI样式
*4.支持js方法在特殊的状态调用重写addEvent和removeEvent方法，在需要的位置调用fireEvent方法执行对应的js方法
*5.支持js调模块扩展出来的方法，编写扩展方法WX_EXPORT_METHOD(@selector(method))扩展
*6.使用WXSDKEngine的+registerComponent:withClass:去注册
js端
1.直接在需要该组件的template中使用
2.如果组件需要传递参数，则可以通过:key=value的形式传递，方法@methodName=JSmethod,实现js方法
3.调用扩展方法：this.$refs.refName.methodName
```
iOS模块`Module`扩展步骤
```
iOS端
*1.创建自定义的模块实现<WXModuleProtocol>协议
*2.实现自定义的功能并callback回调给js
*3.使用WX_EXPORT_METHOD或WX_EXPORT_METHOD_SYNC来抛出native方法给js
*4.使用WXSDKEngine的+registerModule:withClass:去注册
js端
1.直接在需要该组件的script中使用weex.requireModule()获得已注册的模块
2.在需要的位置调用注册的方法
```
 * **2.2 weex-android插件扩展方式介绍**

android组件`Component`扩展步骤
```
android端
*1.创建自定义的组件继承WXComponent/WXVContainer类
*2.使用initComponentHostView(context)初始化，initView()方法已过时
*3.现实自定义的UI样式, 通过添加注解@WXComponentProp设置属性
*4.使用instance.fireEvent()方法执行对应的js方法
*5.支持js调模块扩展出来的方法，编写扩展方法@JSMethod(uiThread = false或true)扩展
*6.使用WXSDKEngine的WXSDKEngine.registerComponent()去注册
js端
1.直接在需要该组件的template中使用
2.如果组件需要传递参数，则可以通过:key=value的形式传递，方法@methodName=JSmethod,实现js方法
3.调用扩展方法：this.$refs.refName.methodName
```
android模块`Module`扩展步骤
```
android端
*1.创建自定义的模块继承WXModule
*2.实现自定义的功能并callback回调给js
*3.使用@JSMethod(uiThread = false或true)将自定义的方法抛给js
*4.使用WXSDKEngine的WXSDKEngine.registerModule()去注册
js端
1.直接在需要该组件的script中使用weex.requireModule()获得已注册的模块
2.在需要的位置调用注册的方法
```
 * **2.3 weex-web插件扩展方式介绍**

web组件`Component`扩展步骤
```
js端
1.使用npm install加载weex-vue-render组件
2.使用Weex.registerComponent('name', component); 注册组件
3.在webpack.config中使用weex.install(components)加载所有组件
4.调用扩展方法：<name></name>
```
web组件`Module`扩展步骤
```
js端
1. 使用npm install加载weex-vue-render组件
2. 使用Weex.registerModule('name', module)
3. 在webpack.config中使用weex.install(modules)加载所有组件
4.调用扩展方法：const moduleName = weex.requireModule('ModuleName'); 调用模块方法moduleName.xx()
```
######3.Weex 交互（有相关经验的小伙伴可忽略）
* **3.1 Weex-web和native交互（iOS使用WKWebView，android使用WebView）**
```
Weex-web端
1.Weex-web调用native提供的方法：
iOS：webkit.messageHandlers.weexCallHandler.postMessage(data)
android：            window.dianwoda.weexCallHandler(data);
2.解决方案：使用callHandler的调用方式来实现
参数：
a. name：调用native的方法名，即约定的具体需要调用native的什么方法，例：setHeaderTitle（设置header的title）
b. params：传给native端的参数，例：{'title': '这是商品详情页的标题'}
c. onSuccess：调用native方法成功后的回调方法名
d. onFail：调用native的方法名，即约定的具体需要调用native的什么方法，例：setHeaderTitle（设置header的title）
3.问题：由webpack打包生成的Weex-web的vuejs文件的methods方法不在window下native无法调用到，
解决方案：
var hybrid = {}
if (weex.config.env.platform === 'Web') {
    window.Hybrid = hybrid
}
将方法绑到window下面才能调到
iOS端
1.iOS调用Weex-web的方法回传数据：[webView evaluateJavaScript:jsStr completionHandler:^(id _Nullable d, NSError * _Nullable error) {}];
android端
1.. android调用Weex-web的方法回传数据：通过getSettings()获得WebSettings，然后用setJavaScriptEnabled()调用Js
```
* **3.2 Weex和iOS交互**
```
1.Weex调用iOS的方法
方式1：通过iOS注册的Module方式来抛出方法供weex调用
2.iOS调用weex的方法
方式1：拿到weex页面的instanceId通过fireEvent方法调用weex方法
如：[[WXSDKManager bridgeMgr] fireEvent:weexInstance.instanceId ref:WX_SDK_ROOT_REF type:type params:params domChanges:nil];
方式2：在组件中使用fireEvent方法来调用weex方法
方式3：使用fireGlobalEvent来通知调用weex方法

注：fireEvent方式是通过JSContext实现，fireGlobalEvent是通过通知实现
```
* **3.3 Weex和android交互**
```
1.Weex调用android的方法
方式1：通过android注册的Module方式来抛出方法供weex调用
2. android调用weex的方法
方式1：拿到weex页面的instance通过fireEvent(getRef(), numClick, data, null);方法来调用weex方法
方式3：使用fireGlobalEvent来广播调用weex方法

注：fireEvent方式是通过webView调用js实现，fireGlobalEvent是通过广播实现
```
######4.Weex SVG实践
* **4.1 Weex-web SVG扩展**

SVG可在Weex-web上直接使用：

* 矩形 `<rect>`
```
<svg width="100%" height="100%" version="1.1" xmlns="http://www.w3.org/2000/svg">
    <rect x="20" y="20" rx="10" ry="10" width="300" height="100" style="fill:rgb(0,0,255);stroke-width:1;stroke:rgb(0,0,0);fill-opacity:0.1;stroke-opacity:0.9;opacity:0.9"/>
</svg>

代码解释：
rect 元素的 width 和 height 属性可定义矩形的高度和宽度
style 属性用来定义 CSS 属性
CSS 的 fill 属性定义矩形的填充颜色（rgb 值、颜色名或者十六进制值）
CSS 的 stroke-width 属性定义矩形边框的宽度
CSS 的 stroke 属性定义矩形边框的颜色
x 属性定义矩形的左侧位置（例如，x="0" 定义矩形到浏览器窗口左侧的距离是 0px）
y 属性定义矩形的顶端位置（例如，y="0" 定义矩形到浏览器窗口顶端的距离是 0px）
CSS 的 fill-opacity 属性定义填充颜色透明度（合法的范围是：0 - 1）
CSS 的 stroke-opacity 属性定义笔触颜色的透明度（合法的范围是：0 - 1
CSS 的 opacity 属性定义整个元素的透明值（合法的范围是：0 - 1）
rx 和 ry 属性可使矩形产生圆角。
```
![alt](http://tech.dianwoda.com/content/images/2018/01/6344593-98ba3f0bf118f590.png)

* 圆形 `<circle>`
```
<svg width="100%" height="100%" version="1.1"
xmlns="http://www.w3.org/2000/svg">

<circle cx="100" cy="50" r="40" stroke="black"
stroke-width="2" fill="red"/>

</svg>

代码解释：
cx 和 cy 属性定义圆点的 x 和 y 坐标。如果省略 cx 和 cy，圆的中心会被设置为 (0, 0)。
r 属性定义圆的半径。
```
![alt](http://tech.dianwoda.com/content/images/2018/01/lALPBbCc1UqBVHXMtsz9_253_182-png_620x10000q90g.jpg)
* 椭圆 `<ellipse>`
```
<svg width="100%" height="100%" version="1.1" xmlns="http://www.w3.org/2000/svg">
   <ellipse cx="275" cy="125" rx="100" ry="50" style="fill:rgb(200,100,50);stroke:rgb(0,0,100);stroke-width:2"/>
</svg>


代码解释：
cx 属性定义圆点的 x 坐标
cy 属性定义圆点的 y 坐标
rx 属性定义水平半径
ry 属性定义垂直半径
```
![alt](http://tech.dianwoda.com/content/images/2018/01/6344593-02c89759c68b7120.png)
* 线 `<line>`
```
<svg width="100%" height="100%" version="1.1" xmlns="http://www.w3.org/2000/svg">
 <line x1="0" y1="0" x2="300" y2="300" style="stroke:rgb(99,99,99);stroke-width:2"/>
</svg>

代码解释：
x1 属性在 x 轴定义线条的开始
y1 属性在 y 轴定义线条的开始
x2 属性在 x 轴定义线条的结束
y2 属性在 y 轴定义线条的结束
```
![alt](http://tech.dianwoda.com/content/images/2018/01/6344593-1c467d6d31e085d5.png)
* 折线 `<polyline>`
```
<svg width="100%" height="100%" version="1.1" xmlns="http://www.w3.org/2000/svg">
    <polyline points="20,100 40,60 70,80 100,20" style="fill:white;stroke:red;stroke-width:2"></polyline>
</svg>
代码解释：
points
点集数列。每个数字用空白符、逗号、终止命令或者换行符分隔开。每个点必须包含2个数字，一个是x坐标，一个是y坐标。所以点列表 (0,0), (1,1) 和(2,2)可以写成这样：“0 0, 1 1, 2 2”。路径绘制完后闭合图形，所以最终的直线将从位置(2,2)连接到位置(0,0)。
```
![alt](http://tech.dianwoda.com/content/images/2018/01/6344593-66f0a9449c351415.png)
* 多边形 `<polygon>`
```
<svg width="100%" height="100%" version="1.1" xmlns="http://www.w3.org/2000/svg">
    <polygon points="220,100 300,210 170,250" style="fill:#cccccc;stroke:#000000;stroke-width:1"/>
</svg>

代码解释：
points 属性定义多边形每个角的 x 和 y 坐标
点集数列。每个数字用空白符、逗号、终止命令或者换行符分隔开。每个点必须包含2个数字，一个是x坐标，一个是y坐标。所以点列表 (0,0), (1,1) 和(2,2)可以写成这样：“0 0, 1 1, 2 2”。路径绘制完后闭合图形，所以最终的直线将从位置(2,2)连接到位置(0,0)。
```
![alt](http://tech.dianwoda.com/content/images/2018/01/6344593-a23fe33c6d435891.png)

* 路径 `<path>`
```
           <svg>
                <path d="M 0,304 C 0,304 375,270 750,0" stroke-width="4"  stroke="#3A3A41" fill="#42424F" />
            </svg>

代码解释：
path> 标签用来定义路径。
下面的命令可用于路径数据：
M = moveto
L = lineto
H = horizontal lineto
V = vertical lineto
C = curveto
S = smooth curveto
Q = quadratic Belzier curve
T = smooth quadratic Belzier curveto
A = elliptical Arc
Z = closepath
注释：以上所有命令均允许小写字母。大写表示绝对定位，小写表示相对定位。
```
![alt](http://tech.dianwoda.com/content/images/2018/01/lALPBbCc1UqKDxHNAXDNAtw_732_368-png_620x10000q90g.jpg)

* **4.2 Weex-iOS SVG扩展**
```
思路：根据svg api规范在iOS端通过 Core Graphics库来实现一套iOS-svg标准的组件库，并通过weex Component的方式把这个组件库暴露给weex使用。
```
4.2.1.使用Core Graphics来定义符合svg api规范的各种view，如线，圆形，矩形等等。
```
Core Graphics的使用介绍
这是一个绘图专用的API族，它经常被称为QuartZ或QuartZ 2D。Core Graphics是iOS上所有绘图功能的基石，包括UIKit。
基本概念：
** 1.绘图需要 CGContextRef**
CGContextRef即图形上下文。可以这么理解，我们绘图是需要一个载体或者说输出目标，它用来显示绘图信息，并且决定绘制的东西输出到哪个地方。可以形象的比喻context就像一个“画板”，我们得把图形绘制到这个画板上。所以，绘图必须要先有context。

** 2.怎么拿到context？**
第一种方法是利用cocoa为你生成的图形上下文。当你子类化了一个UIView并实现了自己的drawRect：方法后，一旦drawRect：方法被调用，Cocoa就会为你创建一个图形上下文，此时你对图形上下文的所有绘图操作都会显示在UIView上。
即：重写UIView的drawRect方法，在该方法里便可得到context；

第二种方法就是创建一个图片类型的上下文。调用UIGraphicsBeginImageContextWithOptions函数就可获得用来处理图片的图形上下文。利用该上下文，你就可以在其上进行绘图，并生成图片。调用UIGraphicsGetImageFromCurrentImageContext函数可从当前上下文中获取一个UIImage对象。记住在你所有的绘图操作后别忘了调用UIGraphicsEndImageContext函数关闭图形上下文。
即：调用UIGraphicsBeginImageContextWithOptions方法得到context；
```
Core Graphics的使用步骤：
```
1.先在drawRect方法中获得上下文context；
2.绘制图形（线，图形，图片等）；
3.设置一些修饰属性；
4.渲染到上下文，完成绘图。
```
代码：
```
#import "CustomView.h"

@implementation CustomView

- (void)drawRect:(CGRect)rect
{
    // 1.获取上下文
    CGContextRef ctx = UIGraphicsGetCurrentContext();
    
    // --------------------------实心圆
    
    // 2.画图
    CGContextAddEllipseInRect(ctx, CGRectMake(10, 10, 50, 50));
    [[UIColor greenColor] set];
    
    // 3.渲染
    CGContextFillPath(ctx);
    
    
    
    // --------------------------空心圆
    
    CGContextAddEllipseInRect(ctx, CGRectMake(70, 10, 50, 50));
    [[UIColor redColor] set];
    CGContextStrokePath(ctx);
    
    
    
    // --------------------------椭圆
    //画椭圆和画圆方法一样，椭圆只是设置不同的长宽
    CGContextAddEllipseInRect(ctx, CGRectMake(130, 10, 100, 50));
    [[UIColor purpleColor] set];
    CGContextFillPath(ctx);
    
    
    
    // --------------------------直线
    CGContextMoveToPoint(ctx, 20, 80); // 起点
    CGContextAddLineToPoint(ctx, self.frame.size.width-10, 80); //终点
//    CGContextSetRGBStrokeColor(ctx, 0, 1.0, 0, 1.0); // 颜色
    [[UIColor redColor] set]; // 两种设置颜色的方式都可以
    CGContextSetLineWidth(ctx, 2.0f); // 线的宽度
    CGContextSetLineCap(ctx, kCGLineCapRound); // 起点和重点圆角
    CGContextSetLineJoin(ctx, kCGLineJoinRound); // 转角圆角
    CGContextStrokePath(ctx); // 渲染（直线只能绘制空心的，不能调用CGContextFillPath(ctx);）
    
    
    
    // --------------------------三角形
    CGContextMoveToPoint(ctx, 10, 150); // 第一个点
    CGContextAddLineToPoint(ctx, 60, 100); // 第二个点
    CGContextAddLineToPoint(ctx, 100, 150); // 第三个点
    [[UIColor purpleColor] set];
    CGContextClosePath(ctx);
    CGContextStrokePath(ctx);
    
    
    
    // --------------------------矩形
    CGContextAddRect(ctx, CGRectMake(20, 170, 100, 50));
    [[UIColor orangeColor] set];
//    CGContextStrokePath(ctx); // 空心
    CGContextFillPath(ctx);
    
    
    
    // --------------------------圆弧
    CGContextAddArc(ctx, 200, 170, 50, M_PI, M_PI_4, 0);
    CGContextClosePath(ctx);
    CGContextFillPath(ctx);
    
    
    // --------------------------文字
    NSString *str = @"你在点我达，我在点我达";
    NSMutableDictionary *dict = [NSMutableDictionary dictionary];
    dict[NSForegroundColorAttributeName] = [UIColor whiteColor]; // 文字颜色
    dict[NSFontAttributeName] = [UIFont systemFontOfSize:14]; // 字体
    
    [str drawInRect:CGRectMake(20, 250, 300, 30) withAttributes:dict];
    

    // --------------------------图片
    UIImage *img = [UIImage imageNamed:@"yingmu"];
//    [img drawAsPatternInRect:CGRectMake(20, 280, 300, 300)]; // 多个平铺
//    [img drawAtPoint:CGPointMake(20, 280)]; // 绘制到指定点，图片有多大就显示多大
    [img drawInRect:CGRectMake(20, 280, 80, 80)]; // 拉伸
}
```
4.2.2.将使用Core Graphics来定义符合svg api规范的各种view通过Component扩展给weex。

例如：给weex扩展划线的api

**1.使用Core Graphics定义一条线**
```
@implementation WXSVGLine

- (void)setX1:(CGFloat)x1
{
    if (x1 == _x1) {
        return;
    }
    [self invalidate];
    _x1 = x1;
}

- (void)setY1:(CGFloat)y1
{
    if (y1 == _y1) {
        return;
    }
    [self invalidate];
    _y1 = y1;
}

- (void)setX2:(CGFloat)x2
{
    if (x2 == _x2) {
        return;
    }
    [self invalidate];
    _x2 = x2;
}

- (void)setY2:(CGFloat)y2
{
    if (y2 == _y2) {
        return;
    }
    [self invalidate];
    _y2 = y2;
}

- (CGPathRef)getPath:(CGContextRef)context
{
    [self setBoundingBox:CGContextGetClipBoundingBox(context)];
    CGMutablePathRef path = CGPathCreateMutable();
    CGFloat x1 = self.x1;//[self getWidthRelatedValue:self.x1];
    CGFloat y1 = self.y1;//[self getHeightRelatedValue:self.y1];
    CGFloat x2 = self.x2;//[self getWidthRelatedValue:self.x2];
    CGFloat y2 = self.y2;//[self getHeightRelatedValue:self.y2];
    CGPathMoveToPoint(path, nil, x1, y1);
    CGPathAddLineToPoint(path, nil, x2, y2);
    
    return (CGPathRef)CFAutorelease(path);
}

@end
```

2.通过Component把线扩展给weex使用
```
@implementation WXSVGLineComponent
{
    NSString *_x1;
    NSString *_y1;
    NSString *_x2;
    NSString *_y2;
}

#pragma mark -
#pragma mark - override methods
- (instancetype)initWithRef:(NSString *)ref
                       type:(NSString*)type
                     styles:(nullable NSDictionary *)styles
                 attributes:(nullable NSDictionary *)attributes
                     events:(nullable NSArray *)events
               weexInstance:(WXSDKInstance *)weexInstance
{
    self = [super initWithRef:ref type:type styles:styles attributes:attributes events:events weexInstance:weexInstance];
    if (self) {
        _x1 = attributes[@"x1"] ? : @"0";
        _x2 = attributes[@"x2"] ? : @"0";
        _y1 = attributes[@"y1"] ? : @"0";
        _y2 = attributes[@"y2"] ? : @"0";
    }
    
    return self;
}


#pragma mark -
#pragma mark -
- (WXSVGRenderable *)node
{
    WXSVGLine *lineView = [WXSVGLine new];
    lineView.x1 = [WXConvert WXPixelType:_x1 scaleFactor:self.weexInstance.pixelScaleFactor];
    lineView.y1 = [WXConvert WXPixelType:_y1 scaleFactor:self.weexInstance.pixelScaleFactor];
    lineView.x2 = [WXConvert WXPixelType:_x2 scaleFactor:self.weexInstance.pixelScaleFactor];
    lineView.y2 = [WXConvert WXPixelType:_y2 scaleFactor:self.weexInstance.pixelScaleFactor];
    [self syncViewAttributes:lineView];
    return lineView;
}
    
- (void)updateAttributes:(NSDictionary *)attributes
{
    WXSVGLine *lineView = (WXSVGLine *)self.view;
    if (attributes[@"x1"]) {
        _x1 = attributes[@"x1"];
        lineView.x1 = [WXConvert WXPixelType:_x1 scaleFactor:self.weexInstance.pixelScaleFactor];
    }
    if (attributes[@"x2"]) {
        _x2 = attributes[@"x2"];
        lineView.x2 = [WXConvert WXPixelType:_x2 scaleFactor:self.weexInstance.pixelScaleFactor];
    }
    if (attributes[@"y1"]) {
        _y1 = attributes[@"y1"];
        lineView.y1 = [WXConvert WXPixelType:_y1 scaleFactor:self.weexInstance.pixelScaleFactor];
    }
    if (attributes[@"y2"]) {
        _y2 = attributes[@"y2"];
        lineView.y2 = [WXConvert WXPixelType:_y2 scaleFactor:self.weexInstance.pixelScaleFactor];
    }
    [super updateAttributes:attributes];
}

@end
```
 * **4.3 Weex-android SVG扩展**
```
思路：根据svg api规范在android端通过 graphics库来实现一套android-svg标准的组件库，并通过weex Component的方式把这个组件库暴露给weex使用。
```
4.3.1.使用graphics来定义符合svg api规范的各种view，如线，圆形，矩形等等。
```
Android画图--同样有三个基本对象：Color，Paint，Canvas
它们都位于android.graphics画图包下面。
1.Color： 颜色对象，相当于现实生活中的 调料
2.Paint : 画笔对象，相当于现实生活中画图用的 笔。对画笔的一些参数设置是很重要的。
3.Canvas : 画布对象，相当于现实生活中画图用的画纸或者画布
```
* 使用Canvas绘制点
```
void drawPoint(float x, float y, Paint paint)
画点，参数一水平x轴，参数二垂直y轴，第三个参数为Paint对象。
void drawPoints (float[] pts, Paint paint)
绘制一组点，坐标位置由float数组指定
```
* 使用Canvas绘制直线
```
void drawLine(float startX, float startY, float stopX, float stopY, Paint paint)
画线，参数一起始点的x轴位置，参数二起始点的y轴位置，参数三终点的x轴水平位置，参数四y轴垂直位置，最后一个参数为Paint对象。
void drawLines (float[] pts, Paint paint)
绘制一组线 ，坐标位置由float数组指定
```
* 使用Canvas绘制矩形
```
void drawRect (float left, float top, float right,float bottom, Paint paint)
绘制矩形，四个数值(矩形左上角和右下角两个点的坐标)来确定一个矩形
void drawRect(Rect r, Paint paint)
绘制矩形，参数一为Rect一个区域
void drawRect (RectF rect,Paint paint)
绘制矩形，参数一为RectF 一个区域
对于第二、三种方式，我们需要传入一个Rect或RectF
Rect rect = new Rect(100,100,800,400);
canvas.drawRect(rect,mPaint);
RectF rectF = new RectF(100.1f,100.1f,800.1f,400.1f);
canvas.drawRect(rectF,mPaint);
这两者的主要区别是：精度不一样。Rect是使用int类型作为数值，RectF是使用float类型作为数值
```
* 使用Canvas绘制圆形矩形
```
void drawRoundRect(@NonNull RectF rect, float r1, float r2, @NonNull Paint paint)
绘制圆角矩形，rx, ry分别是圆弧的圆心 和 半径，其中圆心用于确定位置，而半径用于确定大小
void drawRoundRect (float left,float top, float right, float bottom, float r1, float r2, Paint paint)
API level 21才添加的
```
* 使用Canvas绘制椭圆
```
void drawOval(@NonNull RectF oval, @NonNull Paint paint)
绘制椭圆
drawOval(float left, float top, float right, float bottom, Paint paint)
```
* 使用Canvas绘制圆
```
drawCircle(float cx, float cy, float radius, @NonNull Paint paint)
绘制圆，cx，cy是圆心坐标，radius是半径，paint是画笔对象
```
* 使用Canvas绘制路径
```
void drawPath(Path path, Paint paint)
绘制一个路径，参数一为Path路径对象
```
Paint
* Paint的基本用法
```
void setARGB(int a, int r, int g, int b)
设置Paint对象颜色，参数一为alpha透明通道

void setAlpha(int a)
设置alpha不透明度，范围为0~255

void setAntiAlias(boolean aa)
是否抗锯齿

void setColor(int color)
设置颜色，这里Android内部定义的有Color类包含了一些常见颜色定义

void setStyle(Style style)
设置画笔样式为描边,画笔样式分三种：
1.Paint.Style.STROKE：描边
2.Paint.Style.FILL_AND_STROKE：描边并填充
3.Paint.Style.FILL：填充

void setFakeBoldText(boolean fakeBoldText)
设置伪粗体文本

void setLinearText(boolean linearText)
设置线性文本

void setTextAlign(Paint.Align align)
设置文本对齐

void setTextScaleX(float scaleX)
设置文本缩放倍数，1.0f为原始

void setTextSize(float textSize)
设置字体大小

void setUnderlineText(boolean underlineText)
设置下划线

setXfermode(Xfermode xfermode)
设置图像混合模式
关于这个方法可以看这个链接了解
http://www.cnblogs.com/tianzhijiexian/p/4297172.html

Rasterizer setRasterizer(Rasterizer rasterizer)
设置光栅化，实际效果并不明显

Typeface setTypeface(Typeface typeface)
设置字体，Typeface包含了字体的类型，粗细，还有倾斜、颜色等。
Typeface字体对象，这个类的作用是获取字体，创建字体，以及设置字体。

PathEffect setPathEffect(PathEffect effect)
设置路径效果

Shader setShader(Shader shader)
设置阴影
```
Color
* Color的基本用法系统颜色
```
可以通过 Color.颜色名，来获取颜色，应为是静态的，返回一个整数值
有以下几个: BLACK(黑色),BLUE(蓝色),CYAN(青色),GRAY(灰色),GREEN(绿色),RED(红色),WRITE(白色),YELLOW(黄色)等
Button btn = (Button) findViewById(R.id.btn);
btn.setBackgroundColor(Color.BLUE);
```
* Color的基本用法在XML中自定义颜色
```
<!--?xml version=1.0 encoding=utf-8?-->
<resources>
    <color name="mycolor">#748751</color>
</resources>
//在java代码中的使用
int mycolor = getResources().getColor(R.color.mycolor);
Button btn = (Button) findViewById(R.id.btn);
btn.setBackgroundColor(mycolor);
```
* Color的基本用法在XML中自定义颜色
```
//若已知颜色
int mycolor = 0xff123456;0x代表16进制FF代表透明度
//或调用静态的 argb方法，可以调出个性的颜色
int myColor=Color.argb(0xff, 255, 255, 255)
argb()方法的参数依次为透明度。后三个为红,绿,蓝的大小。范围都是[0-255]，0至255 颜色依次加深
```

4.3.2.将使用android.graphics来定义符合svg api规范的各种view通过Component扩展给weex。

例如：给weex扩展划线的api
**1.使用android.graphics定义一条线**
```
import android.graphics.Canvas;
import android.graphics.Paint;
import android.graphics.Path;

import com.alibaba.weex.svg.ParserHelper;
import com.taobao.weex.WXSDKInstance;
import com.taobao.weex.dom.WXDomObject;
import com.taobao.weex.ui.ComponentCreator;
import com.taobao.weex.ui.component.WXComponent;
import com.taobao.weex.ui.component.WXComponentProp;
import com.taobao.weex.ui.component.WXVContainer;

import java.lang.reflect.InvocationTargetException;



public class WXSvgLine extends WXSvgPath {
  private String mX1;

  private String mY1;

  private String mX2;

  private String mY2;

  public WXSvgLine(WXSDKInstance instance, WXDomObject dom, WXVContainer parent) {
    super(instance, dom, parent);
  }


  @WXComponentProp(name = "x1")
  public void setX1(String x1) {
    mX1 = x1;
  }

  @WXComponentProp(name = "y1")
  public void setY1(String y1) {
    mY1 = y1;
  }

  @WXComponentProp(name = "x2")
  public void setX2(String x2) {
    mX2 = x2;
  }

  @WXComponentProp(name = "y2")
  public void setY2(String y2) {
    mY2 = y2;
  }

  @Override
  public void draw(Canvas canvas, Paint paint, float opacity) {
    mPath = getPath(canvas, paint);
    super.draw(canvas, paint, opacity);
  }

  @Override
  protected Path getPath(Canvas canvas, Paint paint) {
    Path path = new Path();
    float x1 = ParserHelper.fromPercentageToFloat(mX1, mCanvasWidth, 0, mScale);
    float y1 = ParserHelper.fromPercentageToFloat(mY1, mCanvasHeight, 0, mScale);
    float x2 = ParserHelper.fromPercentageToFloat(mX2, mCanvasWidth, 0, mScale);
    float y2 = ParserHelper.fromPercentageToFloat(mY2, mCanvasHeight, 0, mScale);

    path.moveTo(x1, y1);
    path.lineTo(x2, y2);
    return path;
  }

  public static class Creator implements ComponentCreator {
    public Creator() {
    }

    public WXComponent createInstance(WXSDKInstance instance, WXDomObject node, WXVContainer parent) throws IllegalAccessException, InvocationTargetException, InstantiationException {
      return new WXSvgLine(instance, node, parent);
    }
  }
}
```

######5.SVG编辑工具介绍 
* **5.1 SVGSUS**

**SVGSUS功能特色：**

* 支持导入 SVG 图标包，管理方便
* 可直接把图标拖至 AI/PS/SKETCH 等工具上使用
* 图标搜索功能
* 可输出不同终端用的 SVG 代码（如：Web/Android/iOS）
* 下面我们来看看这个 svgsus 工具的使用体验。

**导入图标包**
![alt](http://tech.dianwoda.com/content/images/2018/01/svgsus-demo-1.gif)
**使用图标，用户可以很方便的从软件把图标拖到 AI、PS、Sketch 等设计工具上直接使用。**
![alt](http://tech.dianwoda.com/content/images/2018/01/svgsus-demo-2.gif)
**转换为 SVG 图标代码**
```
svgsus 工具可以转换种代码格式，包括：Web, iOS, OS X, Android 等。使用方法也超简单：
方法1：单击你需要用的图标，然后在编辑器上 Ctrl + V 即可粘贴代码。
方法2：直接拖动图标到编辑器
方法3：这个有点帅，直接在拖动图标时，经过 svgsus 界面，再拖入编辑器里，就可以生成代码了，请看帅气的 DEMO 吧！
```
![alt](http://tech.dianwoda.com/content/images/2018/01/svgsus-demo-3.gif)
![alt](http://tech.dianwoda.com/content/images/2018/01/svgsus-demo-4.gif)
**从AI、Sketch 设计软件中输出 SVG 图标**
```
首先在 AI 或 Sketch 里复制你要输出的图标，然后在 svgsus 上粘贴一下，最后，在文件夹里粘贴一下，完成！请看帅气的 DEMO 吧！
```
![alt](http://tech.dianwoda.com/content/images/2018/01/svgsus-demo-5.gif)
######6. Weex开发过程中遇到的一些坑总结
**[6.1 Weex 开发过程中遇到的一些坑总结](http://tech.dianwoda.com/2017/12/25/weexshi-yong-guo-cheng-zhong-de-na-xie-keng/)**

结语：以上是笔者对SVG的探索加一点点自己的理解，当然还有很多不足之处，由于公司比较忙没有这么多时间全部写完，有空闲我会慢慢把补充一些原理部分上去，欢迎小伙伴前来Issue/PR,希望能帮助到正在使用weex开发的小伙伴们少走些弯路。



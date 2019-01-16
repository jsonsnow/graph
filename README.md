#### 寄宿图
##### 设置图层的寄宿图
* 1.设置layer的contents属性
* 2.通过用Core Grapics直接绘制寄宿图，-drawRect：方法来自定绘制

##### 自定义绘制简绍
关于自定义绘制，如果UIView检测到-drawRect:方法被调用，它就会为视图分配一个寄宿图，寄宿图的像素尺寸等于视图大小乘以contentScale的值。

-drawRect:调用时机，当视图在屏幕上出现的时候-drawRect方法被自动调用，-drawRect:方法里面的代码利用Core Graphics去绘制一个寄宿图，然后内容就会被缓存起来直到它需要被更新(通常是因为开发者调用了setNeedsDisplay，或者一些影响表现效果的属性值被更改)。事实上底层是CALayer安排了重绘工作和保存了因此产生的图片。

```
1.
override func display(_ layer: CALayer) {
 }
 
 2.
 func draw(_ layer: CALayer, in ctx: CGContext) {
        
 }
    
```
可以在1如直接赋值一个寄宿图，没设置的话则会调用2方法

#### 图层几何学
##### 布局属性

UIView有三个比较重要的布局属性：frame，bounds和center，CALayer对应地叫做frame，bounds和position。为了能清楚区分，图层用了“position”，视图用了“center”，但是他们都代表同样的值。

##### 锚点
我们知道position和center都是相对于父视图的，但把视图的那个点放到position或center上就是锚点来决定的，默认是(0.5,0.5),也就是把视图的中心放到position或center上

##### Hit Testing

CALayer并不关心任何响应链事件，但它有一系列的方法帮你处理事件:-containsPoint和-hitTest:。

-containsPoint: 接受一个在__本图层__坐标系下的CGPoint, 如果这个点在图层 frame范围内就返回YES。 eg:

```
- (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event
{
    //get touch position relative to main view
    CGPoint point = [[touches anyObject] locationInView:self.view];
    //convert point to the white layer's coordinates
    point = [self.layerView.layer convertPoint:point fromLayer:self.view.layer];
    //get layer using containsPoint:
    if ([self.layerView.layer containsPoint:point]) {
        //convert point to blueLayer’s coordinates
        point = [self.blueLayer convertPoint:point fromLayer:self.layerView.layer];
        if ([self.blueLayer containsPoint:point]) {
            [[[UIAlertView alloc] initWithTitle:@"Inside Blue Layer"
                                        message:nil
                                       delegate:nil
                              cancelButtonTitle:@"OK"
                              otherButtonTitles:nil] show];
        } else {
            [[[UIAlertView alloc] initWithTitle:@"Inside White Layer"
                                        message:nil
                                       delegate:nil
                              cancelButtonTitle:@"OK"
                              otherButtonTitles:nil] show];
        }
    }
}
```
注意上面坐标系的转换，因为是控制器的view，得到的是self.view下的point,通过 covertPoint来转换，可以一次性转换到blueLayer下，但往往会先判断是否在其父视图中。

上述判断可以直接利用-hitTest:方法

```
- (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event
{
    //get touch position
    CGPoint point = [[touches anyObject] locationInView:self.view];
    //get touched layer
    CALayer *layer = [self.layerView.layer hitTest:point];
    //get layer using hitTest
    if (layer == self.blueLayer) {
        [[[UIAlertView alloc] initWithTitle:@"Inside Blue Layer"
                                    message:nil
                                   delegate:nil
                          cancelButtonTitle:@"OK"
                          otherButtonTitles:nil] show];
    } else if (layer == self.layerView.layer) {
        [[[UIAlertView alloc] initWithTitle:@"Inside White Layer"
                                    message:nil
                                   delegate:nil
                          cancelButtonTitle:@"OK"
                          otherButtonTitles:nil] show];
    }
}
```

#### 视觉效果

##### 图层蒙板

CALayer有一个属性叫做mask，为CALayer类型，有和其他图层一样的绘制和布局属性。类似一个子图层，但mask图层定义了父图层的部分可见区域。

mask图层的Color属性是无关紧要的，真正重要的是图层的轮廓。
mask不限于静态图，任何有图层构成的都可以作为mask属性，mask可以通过代码甚至是动画实时生成。

mask结合CAShapeLayer设置不规则圆角

```
//define path parameters
CGRect rect = CGRectMake(50, 50, 100, 100);
CGSize radii = CGSizeMake(20, 20);
UIRectCorner corners = UIRectCornerTopRight | UIRectCornerBottomRight | UIRectCornerBottomLeft;
//create path
UIBezierPath *path = [UIBezierPath bezierPathWithRoundedRect:rect byRoundingCorners:corners cornerRadii:radii];
shape.path = path.CGpath;
self.layer.mask = shape;
```

##### 组透明
UIView有一个alpha的属性来确定视图的透明度。 CALayer有一个等同的属性叫opacity,这两个属性都是影响子层级的。

透明度混合。当显示一个50%透明度的图层时，图层的每个像素都会一半显示自己的颜色，另一半显示图层下面的颜色。

当设置了一个图层的透明度，你希望它包含的整个图层像一个整体一样的透明效果，可以通过设置info,plist文字中的UIViewGroupOpcity为YES,但这个属性会影响整个应用

另一个方法是设置CALayer的一个叫做shouldRastersize属性，来实现组透明的效果。如果它被设置为YEW， 在应用透明度之前，图层及其子图层都会被整合成一个整体的图片，这样就没有透明度混合的问题。(该属性和rasterizationScale配套使用)

```
 button2.layer.shouldRasterize = YES;
 button2.layer.rasterizationScale = [UIScreen mainScreen].scale;
```

#### 隐式动画

##### 事务
事务实际上是Core Animation来包含一系列动画集合的机制，任何用指定事务去改变可以做动画的图层属性都不会立刻发送变化，而是当事务一旦提交的时候开始用一个动画过度到新的值。

任何可以做动画的图层属性都会被添加到栈顶的事务，事务可以用+begin和+commit分别来入栈或者出栈。

Core Animation在每个run loop周期中自动开始一次新的事务(run loop是iOS负责收集用户输入，处理定时器或者网络事件并且重新绘制屏幕的东西)，即使不显示的用[CATransaction begin]开始一次事务，任何在一次run loop循环中属性的改变都会被集中起来，然后做一次0.25秒的动画。

##### 隐式动画的实现

当CALayer的属性被修改时候，它会调用-actionForKey: 方法，传递属性的名称。分为如下几步

* 图层首先检测它是否有委托，并且是否实现CALayerDelegate协议指定的actionForLayer: forKey 方法。如果有，直接调用并返回结果。
* 如果没有委托，或者委托没有实现-actionForLayer: forKey 方法，图层接着检测包含属性名称对应行为映射的actions字典。
* 如果actions字典没有包含对应的属性，那么图层接着在它的style字典接着搜索属性名。
* 最后，如果在style里面也找不到对应的行为，那么图层将会将调用定义了每个属性的标志行为-defaultActionForKey:方法。

-actionForKey:要么返回空（将不会有动画发生），要么是CAAction协议对应的对象，最后CALayer拿这个结果去对先前和当前值做动画。

UIKit禁止隐试动画：每个UIView对它关联的图层都扮演了一个委托，并且提供了-actionForLayer: forKey的实现方法。当不在一个动画快的实现中，UIView对所有图层行为返回nil,但是在动画block范围内，它就返回了一个非空值。

##### 呈现层/呈现树
呈现层代表了在任何时刻当前外观效果，呈现图层仅仅当图层首次被提交的时候创建。




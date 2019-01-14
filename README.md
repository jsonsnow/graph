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

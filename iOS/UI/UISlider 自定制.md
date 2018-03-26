# UISlider 自定制

UISlider 是 UIKit 中较为常用的控件，虽然不如 UIButton 和 UIImageView 那样被人们频繁的使用，也不如 UITableView 和 UICollectionView 那样花式繁多，但在某些特定的场合，它还是能派上一定的用场的。接下来让我们看一下它的高级的用法。

## UISlider 需要子类重写才能使用的 API

在使用 UISlider 的时候，经常会有一些比较常见的设计要求，诸如让 UISlider 的进度条高一些，或者是让滑块的大小大一些等。

首先需要声明的是，以下的四个 API 都是用来供子类重写的，无需你直接调用即可生效。

## 设置 UISlider 的高度

```
- (CGRect)trackRectForBounds:(CGRect)bounds;
```

这个方法是用来返回 UISlider 上的进度条的尺寸的，不过通常情况下，它的长度被我们已经设置好了，真正需要修改的只有高度。

```
- (CGRect)trackRectForBounds:(CGRect)bounds {
    bounds = [super trackRectForBounds:bounds]; // 必须通过调用父类的trackRectForBounds 获取一个 bounds 值，否则 Autolayout 会失效，UISlider 的位置会跑偏。
    return CGRectMake(bounds.origin.x, bounds.origin.y, bounds.size.width, h); // 这里面的h即为你想要设置的高度。
}
```

## 设置滑块可触摸范围的大小

有的时候，设计会给我们一个图片让我们来把滑块变得大一些，我们调用了 UISlider 的方法把图片给了滑块。但是，测试又会跟我们说，滑块这么大，怎么可触摸的范围还是那么小？这时，就可以用到下面这个 API 了~

```
- (CGRect)thumbRectForBounds:(CGRect)bounds
                   trackRect:(CGRect)rect
                       value:(float)value
```

其中，bounds 是滑块的大小，rect 是进度条的尺寸，也就是第一个 API 的返回值，而 value 则是 UISlider 当前的值。

```
- (CGRect)thumbRectForBounds:(CGRect)bounds trackRect:(CGRect)rect value:(float)value
{
    bounds = [super thumbRectForBounds:bounds trackRect:rect value:value]; // 这次如果不调用的父类的方法 Autolayout 倒是不会有问题，但是滑块根本就不动~
    return CGRectMake(bounds.origin.x, bounds.origin.y, w, h); // w 和 h 是滑块可触摸范围的大小，跟通过图片改变的滑块大小应当一致。
}
```

### 设置进度条划过的和未划过的图片的尺寸

```
- (CGRect)maximumValueImageRectForBounds:(CGRect)bounds; // 设置滑块右侧进度条的尺寸
- (CGRect)minimumValueImageRectForBounds:(CGRect)bounds; // 设置滑块左侧进度条的尺寸
```

这两个方法我试过了，可是没什么反应……可能是因为我的进度条不是用图片设置的原因吧。

> 如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~
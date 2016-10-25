---
title: Runtime实战之定制TabBarItem大小
date: 2016-05-11 22:17:26
categories: iOS
tags:
---
本篇blog主要讲解如何定制TabBarItem的大小，最终实现AppStore各大主流APP TabBarItem超出TabBar的效果。
<!--more-->

# 方案一：UIEdgeInsets
**适用场景：** <br>

- 适合APP的TabBarItemImage的图片资源放在本地
- 图片超出tabbar的高度，需移动其位置，来进行适应

**弊端：** <br>

若在本地配置好后，tabbar的图片就不能改动了，若tabbar的图片来自服务端，且不停的切换图片的大小，以上则很难满足。若有此方面的需求请看方案二。

**实现：** <br>

` [tabbarItem setImageInsets:UIEdgeInsetsMake(<#CGFloat top#>, <#CGFloat left#>, <#CGFloat bottom#>, <#CGFloat right#>)]` 

注：图片太大超出tabbar时，系统并不会调整image和title的位置，你需要根据图片的高度，计算出需要往上移动的高度，然后设置top和bottom属性即可。切记top = - bottom，否则image将会被拉伸或者被压缩。


# 方案二：Runtime
利用runtime的话相对方案一来说要比较复杂一点，但其灵活度比较高，我们能够根据服务端所给的image来动态的变化TabBarItem的大小，类似像淘宝、京东活动时。思想：主要是利用runtime对UITabBar的layoutSubviews进行重写，然后调整UITabBarItem的位置。另外，当时在做的APP已经有4-5年的历史了，一开始打算自已定制tabbar，发现要改动的还是挺多的，于是就放弃了。做之前也看了前辈iOS程序犭袁的[CYLTabBarController](https://github.com/ChenYilong/CYLTabBarController)，从中也学到了不少思路。


**实现：** <br>
1. 首先我们使用runtime method swizzling交换系统的`- (void)layoutSubviews;` <br>
2. 使用KVC对系统的UITabBarButton、UITabBarSwappableImageView、UITabBarButtonLabel、_UIBadgeView进行捕获 <br> 
3. 拿到控件后我们对其的frame进行计算，判断当前有没有超出tabbar的高度，若超出则进行处理 <br>
4. 再次利用runtime method swizzling交换系统的`- (nullable UIView *)hitTest:(CGPoint)point withEvent:(nullable UIEvent *)event;`使图片超过后也能接受点击 <br>

**代码：** <br>

- method swizzling：

```
static void ExchangedMethod(SEL originalSelector, SEL swizzledSelector, Class class) {
    
    Method originalMethod = class_getInstanceMethod(class, originalSelector);
    Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);
    
    BOOL didAddMethod =
    class_addMethod(class,
                    originalSelector,
                    method_getImplementation(swizzledMethod),
                    method_getTypeEncoding(swizzledMethod));
    
    if (didAddMethod) {
        class_replaceMethod(class,
                            swizzledSelector,
                            method_getImplementation(originalMethod),
                            method_getTypeEncoding(originalMethod));
    }
    else {
        method_exchangeImplementations(originalMethod, swizzledMethod);
    }
}

```
- 计算frame，并对其重新布局

```
UIView *tabBarImageView, *tabBarButtonLabel, *tabBarBadgeView;
        for (UIView *sTabBarItem in childView.subviews) {
            if ([sTabBarItem isKindOfClass:NSClassFromString(@"UITabBarSwappableImageView")]) {
                tabBarImageView = sTabBarItem;
            }
            else if ([sTabBarItem isKindOfClass:NSClassFromString(@"UITabBarButtonLabel")]) {
                tabBarButtonLabel = sTabBarItem;
            }
            else if ([sTabBarItem isKindOfClass:NSClassFromString(@"_UIBadgeView")]) {
                tabBarBadgeView = sTabBarItem;
            }
        }

        NSString *tabBarButtonLabelText = ((UILabel *)tabBarButtonLabel).text;
  
        CGFloat y = CGRectGetHeight(self.bounds) - (CGRectGetHeight(tabBarButtonLabel.bounds) + CGRectGetHeight(tabBarImageView.bounds));
        if (y < 3) {
            if (!tabBarButtonLabelText.length) {
                space -= tabBarButtonLabelHeight;
            }
            
            childView.frame = CGRectMake(childView.frame.origin.x,
                                         y - space,
                                         childView.frame.size.width,
                                         childView.frame.size.height - y + space
                                         );
        }

```

- 让图片超出部分也能响应点击事件

```
- (UIView *)s_hitTest:(CGPoint)point withEvent:(UIEvent *)event {
    if (!self.clipsToBounds && !self.hidden && self.alpha > 0) {
        UIView *result = [super hitTest:point withEvent:event];
        if (result) {
            return result;
        }
        else {
            for (UIView *subview in self.subviews.reverseObjectEnumerator) {
                CGPoint subPoint = [subview convertPoint:point fromView:self];
                result = [subview hitTest:subPoint withEvent:event];
                if (result) {
                    return result;
                }
            }
        }
    }
    return nil;
}

```
# 注意事项

- 在给tabbar设置图片的时候一定要设置图片的`renderingMode`，否则就会出现下图中图片丢失的现象
- UITabBarButton被修改frame之后，仅有UITabBarSwappableImageView能够响应点击事件，不过我们能够在UITabBar的`- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(nullable UIEvent *)event;`方法中捕获到
- 当适配图片后不要忘记适配`_UIBadgeView`的frame


# 效果图
- 正常中间超出 <br>
![部分超出](http://7xq5ax.com1.z0.glb.clouddn.com/tabbar_image_render.png)<br>
- 做活动时全部超出 <br>
![全部超出](http://7xq5ax.com1.z0.glb.clouddn.com/tabbar_more_all.png)<br>
- 图片丢失 <br>
![图片丢失](http://7xq5ax.com1.z0.glb.clouddn.com/tabbar_more.png)<br>
- UIBadgeView <br>
![](http://7xq5ax.com1.z0.glb.clouddn.com/badgevalue.png)

> 感谢大家花费时间来查看这篇blog，需要下载demo的同学请猛戳[Git](https://github.com/PanXianyue/BlogDemo)。
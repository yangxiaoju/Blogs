# 键值编码扩展

Core Animation 扩展了 NSKeyValueCoding 协议，因为它与 [CAAnimation](https://developer.apple.com/documentation/quartzcore/caanimation) 和 [CALayer](https://developer.apple.com/documentation/quartzcore/calayer) 类有关。 该扩展添加了某些键的默认值，扩展了包装约定，并为 [CGPoint](https://developer.apple.com/documentation/coregraphics/cgpoint)，[CGRect](https://developer.apple.com/documentation/coregraphics/cgrect)，[CGSize](https://developer.apple.com/documentation/coregraphics/cgsize) 和 [CATransform3D](https://developer.apple.com/documentation/quartzcore/catransform3d) 类型添加了 key path 支持。

## 键值编码兼容的容器类

CAAnimation 和 CALayer 类是键值编码兼容的容器类，这意味着你可以为任意键设置值。 即使关键字 someKey 不是 CALayer 类的声明属性，仍然可以为它设置一个值，如下所示：

```
[theLayer setValue:[NSNumber numberWithInteger:50] forKey:@"someKey"];
```

你还可以检索任意键的值，例如你将检索其他键路径的值。 例如，要检索以前设置的 someKey 路径的值，可以使用以下代码：

```
someKeyValue=[theLayer valueForKey:@"someKey"];
```

> OS X 注意：CAAnimation 和 CALayer 类会自动存档您为这些类的实例设置的任何其他密钥，以支持 [NSCoding](https://developer.apple.com/documentation/foundation/nscoding) 协议。

## 默认值支持

Core Animation 为键值编码增加了一个约定，从而类可以为没有设置值的键提供默认值。 CAAnimation 和 CALayer 类使用 defaultValueForKey: 类方法支持此约定。

要为键提供默认值，请创建所需类的子类并覆盖其 defaultValueForKey: 方法。 此方法的实现应该检查关键参数并返回适当的默认值。 清单C-1显示了为 masksToBounds 属性提供默认值的图层对象的 defaultValueForKey: 方法的示例实现。

列表 C-1 defaultValueForKey的示例实现：
```
+ (id)defaultValueForKey:(NSString *)key
{
    if ([key isEqualToString:@"masksToBounds"])
         return [NSNumber numberWithBool:YES];
 
    return [super defaultValueForKey:key];
}
```

## 包装约定

如果键的数据由标量值或 C 数据结构组成，则必须将该类型包含在对象中，然后才能将其分配给图层。 同样，在访问该类型时，你必须检索一个对象，然后使用相应类的扩展名打开适当的值。 表C-1列出了常用的 C 类型和用于包装它们的 Objective-C 类。

表 C-1 C 类型的封装类
| C type | Wrapping class |
| - | - |
| [CGPoint](https://developer.apple.com/documentation/coregraphics/cgpoint) | [NSValue](https://developer.apple.com/documentation/foundation/nsvalue) |
| [CGSize](https://developer.apple.com/documentation/coregraphics/cgsize) | NSValue |
| [CGRect](https://developer.apple.com/documentation/coregraphics/cgrect) | NSValue |
| [CATransform3D](https://developer.apple.com/documentation/quartzcore/catransform3d) | NSValue |
| [CGAffineTransform](https://developer.apple.com/documentation/coregraphics/cgaffinetransform) | [NSAffineTransform](https://developer.apple.com/documentation/foundation/nsaffinetransform) (OS X only) |

## 结构的 key path 支持

CAAnimation 和 CALayer 类允许你使用 key path 访问所选数据结构的字段。 此功能是指定要进行动画处理的数据结构的字段的便捷方式。 你还可以将这些约定与 setValue:forKeyPath: 和 valueForKeyPath: 方法一起使用来设置和获取这些字段。

### CATransform3D key path

你可以使用增强型 key path 支持为包含 CATransform3D 数据类型的属性检索特定的转换值。 要指定图层变换的完整 key path，可以使用字符串值 transform 或 sublayerTransform，然后使用表 C-2 中的一个字段 key path。 例如，要指定图层z轴周围的旋转因子，你需要指定关键路径 transform.rotation.z。

表 C-2 转换字段值键路径
![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/UI/Core%20Animation%20%E7%BC%96%E7%A8%8B%E6%8C%87%E5%8D%97/Core%20Animation%20%E7%BC%96%E7%A8%8B%E6%8C%87%E5%8D%97%EF%BC%88%E5%8D%81%E4%B8%80%EF%BC%89%EF%BC%9A%E9%99%84%E5%BD%95%20C%20%E9%94%AE%E5%80%BC%E7%BC%96%E7%A0%81%E6%89%A9%E5%B1%95/C-2.png?raw=true)

以下示例显示如何使用 [setValue:forKeyPath:](https://developer.apple.com/documentation/objectivec/nsobject/1418139-setvalue) 方法修改图层。 该示例将x轴的平移因子设置为10个点，导致图层沿指定的轴移动该量。

```
[myLayer setValue:[NSNumber numberWithFloat:10.0] forKeyPath:@"transform.translation.x"];
```

> 注意：使用 key path 设置值与使用 Objective-C 属性设置值不同。 你不能使用属性符号来设置变换值。 你必须使用带有上述关键路径字符串的 [setValue:forKeyPath:](https://developer.apple.com/documentation/objectivec/nsobject/1418139-setvalue) 方法。

### CGPoint key path

如果给定属性的值是 CGPoint 数据类型，则可以将表C-3中的一个字段名称附加到该属性以获取或设置该值。 例如，要更改图层位置属性的 x 分量，可以写入 key path 位置。

表 C-3 CGPoint 数据结构字段

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/UI/Core%20Animation%20%E7%BC%96%E7%A8%8B%E6%8C%87%E5%8D%97/Core%20Animation%20%E7%BC%96%E7%A8%8B%E6%8C%87%E5%8D%97%EF%BC%88%E5%8D%81%E4%B8%80%EF%BC%89%EF%BC%9A%E9%99%84%E5%BD%95%20C%20%E9%94%AE%E5%80%BC%E7%BC%96%E7%A0%81%E6%89%A9%E5%B1%95/C-3.png?raw=true)

### CGSize key path

如果给定属性的值是 CGSize 数据类型，则可以将表 C-4 中的一个字段名称附加到属性以获取或设置该值。

表 C-4 CGSize 数据结构字段

![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/UI/Core%20Animation%20%E7%BC%96%E7%A8%8B%E6%8C%87%E5%8D%97/Core%20Animation%20%E7%BC%96%E7%A8%8B%E6%8C%87%E5%8D%97%EF%BC%88%E5%8D%81%E4%B8%80%EF%BC%89%EF%BC%9A%E9%99%84%E5%BD%95%20C%20%E9%94%AE%E5%80%BC%E7%BC%96%E7%A0%81%E6%89%A9%E5%B1%95/C-4.png?raw=true)

### CGRect key path

如果给定属性的值是 CGRect 数据类型，则可以将表 C-3 中的以下字段名称附加到属性以获取或设置该值。 例如，要更改图层边界属性的宽度分量，可以写入关键路径 bounds.size.width。

表 C-5 CGRect 数据结构字段
![](https://github.com/yangxiaoju/Blogs/blob/master/iOS/UI/Core%20Animation%20%E7%BC%96%E7%A8%8B%E6%8C%87%E5%8D%97/Core%20Animation%20%E7%BC%96%E7%A8%8B%E6%8C%87%E5%8D%97%EF%BC%88%E5%8D%81%E4%B8%80%EF%BC%89%EF%BC%9A%E9%99%84%E5%BD%95%20C%20%E9%94%AE%E5%80%BC%E7%BC%96%E7%A0%81%E6%89%A9%E5%B1%95/C-5.png?raw=true)

> 如果觉得我写的还不错，请关注我的微博[@小橘爷](http://weibo.com/yanghaoyu0225)，最新文章即时推送~
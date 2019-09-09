# WebPack打包优化二之externals
经过前面的tree-shaking与echarts.dll抽离优化，我们的`reactiveWO.js`文件大小从`818KB`瘦身到`688KB`了，`vendor.dll.js`从`1586KB`到`838KB`。

我很快发现，`moment`与`monment-zone`从npm中打出来的尺寸巨大，占到`443KB`。
查看了一下其cdn版本的大小，发现才`100KB`！这不正常，可以利用webpack的externals属性将其抽取到外部依赖，有如下好处：
- 与DllReferencePlugin相比，externals的好处是：即使更新版本也不必所有项目重新打包。
- 可以充分利用http2的并发请求特性，理论上合并js不再是优化手段，而分离成8~10个才是最优解。
- 可以提升打包速度，webpack碰到externals中配置的依赖会直接跳过。
- 对于echarts这样的依赖，可以从官网定制所需的精减版本（有很多图表类型是用不到的，实践后确实从`748KB`瘦身到了`417KB`，小了整整`331KB`）

将`moment`与`monment-zone`等其它一些包转换为externals外部依赖之后，vendor.dll.js变成了`435KB`，之前是`838KB`，瘦身`403KB`，非常惊人。

### externals能做的不仅如此
下一章我们可以通过它去定制不同路径依赖的处理。
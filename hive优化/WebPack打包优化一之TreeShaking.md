# webpack打包优化一之tree-shaking

tree-shaking是一个老生常谈的问题了，面试时也经常会被问到。

> 它代表的大意就是删除没用到的代码。这样的功能对于构建大型应用时是非常好的，因为日常开发经常需要引用各种库。但大多时候仅仅使用了这些库的某些部分，并非需要全部，此时tree-shaking如果能帮助我们删除掉没有使用的代码，将会大大缩减打包后的代码量。<br>
> tree-shaking在前端界由rollup首先提出并实现，后续webpack在2.x版本也借助于UglifyJS实现了。自那以后，在各类讨论优化打包的文章中，都能看到tree-shaking的身影。<br>

目前webpack发展到了4.x，你的项目里真的有让tree-shaking生效吗？

安装`webpack-bundle-analyzer`一探究竟，发现tree-shaking并没有什么卵用啊，`@core/common`与`@core/components`中没有用到的组件和方法全部被打进来了，导致子项目打包结果高达`800~1100KB`。

是什么原因导致了webpack的tree-shaking没有生效呢？如果说webpack 2.x没生效我还能理解（因为UglifyJs识别不了副作用），可现在是4.x了，不应该这么菜的吧？


我新建了个空项目，使用lerna创建了两个子项目A，B。
让B去引用A项目中导出的模块，打包B项目时发现了一些现象：
- 如果A项目中导出的是纯模块，不依赖第三方库的，那么B项目打包结果中只包含了A项目的引用部分；
- 如果A项目中导出的模块引入了第三方库依赖，那么B项目的打包结果中会包含**这个模块**中所有的引用部分。

拜读了两篇文章，让我找到了一些珠丝马迹
> [Webpack 中的 sideEffects 到底该怎么用？](https://segmentfault.com/a/1190000015689240)
> [Webpack 4的tree-shaking测试](https://github.com/demos-platform/tree-shaking)

确实印证了文章中的一些理论。

给A项目的package.json配上
```javascript
"sideEffects": false
```
果然，打包都按预期的方案执行了，使用了就打进去了，未使用就没打进去，哪怕是引入了第三方依赖。

> webpack4默认是在`production`模式开启了`tree-shaking`的<br>
当组件库未使用外部依赖时，打包时会进行优化（production模式）
当组件内使用了外部依赖时，打包时，会根据外部依赖库的package.json中的sideEffects进行判断，（未导入的组件）如果第三方库为false则不会被打进来，未配置或值为true的则会打进来。<br>
同理，当lerna中的B项目包依赖A项目时，而A项目是组件库时，如果未配置`"sideEffects":false`，则会把es6导出的所有部分都打进来，配置了则会按需打包。<br>
组件库使用sideEffects: false时，webpack会针对性的使用tree-shaking进行优化

那么，给`@core/common`与`@core/components`配置了`"sideEffects":false`之后，确实打包结果已经好很多了。
经过分析，`@core/components`已经没问题了。

可是，总还是有可是。

### 首先，`@core/common`仍然出现了问题
`lodash`还是全进来了，非常大的一个第三方库，压缩后占用了将近`120KB`。
上网搜了一下，发现`loadsh`是用es5方式导出的，从根本上就不支持tree-shaking，看了下它的package.json确实没有`sideEffect`属性。

替换成`lodash-es`就好了。
另外，`lodash-decorators`也不支持tree-shaking，搞笑的是`lodash-decorators-es`虽然支持，但内部依赖了`lodash`，仍然会把整个`lodash`打进来，直接弃了。

最后，又改了一波Moon里的依赖，将此前的
```javascript
import Moon from '@moon';
```
全改成了es6的导入或按路径引入
```javascript
import { ClientType, isIH } from '@moon';
```

### 然后，打包结果里，@core/components的样式没了……
起初怀疑是webpack的Library模式有问题，后来突然想到`sideEffects`这个属性冒似把样式都tree-shaking掉了
```javascript
import './index.less';
```
比如上面这种方式，就会被webpack shaking掉。
要解决这个方法，只需要将`sideEffects`的值由false变为['.less']即可
```javascript
"sideEffects": [
    ".less"
]
```

至此，tree-shaking的优化就差不多了，把能甩的都甩掉了，打包一个空的子项目，只保留页头等公共组件之后从`600KB`变成`240KB`了。
`reactiveWO.js`从`818KB`变为`688KB`
`recurringWO.js`从`753KB`变为`537KB`
`dailyMap.js`从`606KB`变为`268KB`（因为它没有header头）
`project.js`从`712KB`变为`472KB`。
有了一个很大的进步。
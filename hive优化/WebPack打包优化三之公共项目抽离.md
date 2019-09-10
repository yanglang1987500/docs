# webpack打包优化三之公共项目抽离

经过前两轮的优化，我们的项目打包体积已经瘦身了将近一半。拿dailyMap与project来说，起初是`606KB`与`712KB`，目前已经优化到了`268KB`与`472KB`，可见tree-shaking与externals抽离效果确实可见一斑。

然而，目前还存在一个问题：
> 我们的项目是打包到Moon系统里作为独立MPA布署的，每个页面之间都包含着同一份公共代码。拿Header举例，以前都是Include header页去公用，而现在虽然React项目中是共用的Header组件，但实际上打完包出来嵌入到Moon系统中却没有共享。这部分代码都在@core/common项目中，若全部使用到，单独打包出来将近`840KB`！

那么，如何解决这个问题？

### 我想到的第一个解决方案： 独立Header页头
因为页头Header是所有页面共用的部分，一旦发生修改，就牵扯到所有的子项目都需要打包发布！所以先试着将Header独立出来打成Library包，写一个异步加载的组件去加载这个`header.js`。
新建一个子项目: `@client/header`，在打包入口处导出header头，配置`webpack.config.js`文件`output`部分为`libraryTarget: 'var'`
```javascript
output: {
	filename: "header.js",
	publicPath,
	chunkFilename: 'app/header/chunks/[name].[chunkhash:8].js',
	libraryTarget: 'var',
	library: 'ClientHeader'
}
```
此时的header.js打包出来`380KB`，光是一个页头就如此庞大了。于是写了一个异步组件去加载，虽然Header变为异步了，但是页头中有些组件是具备可见权限的用户才需要渲染的。换言之，如果没有权限，这部分js是没有必要加载的。所以将这些内容做成异步加载之后（独立成chunk包），header.js只剩`121KB`了，此时的reactiveWO.js只剩`604KB`了（此前是`688KB`），瘦身`84KB`。recurringWO.js从`537KB`瘦身为`370KB`，project.js从`472KB`瘦身为`238KB`，优化结果更明显。

> 此处还填了一个小坑，开发环境与生产环境要区别对待：开发环境为了便于开发与调试，必须加载实际`Header`组件，而生产环境只需要加载`header.js`即可。


至此，解决了一部分公共代码修改之后需要全部打包发版的问题。但我马上又想到：万一store层有变化怎么办？还是要全部打包发布吗？我们是否能将公共store层也抽离出来呢？抽离出来是做异步加载吗？异步加载之后如何动态放置到`Provider`组件中去呢？一开始没放置会导致取数据报错吗？会导致建立不了观察依赖吗？这些问题困扰了我整整一个周末。直到有一天下班路上突然想到了方案二。
> 搞那么细干嘛，还不如全抽离出来！

### 第二个解决方案：抽离整个@core/common项目
如果说第一个解决方案只是小打小闹的话，那方案二就是大刀阔斧地置之死地而后生。

说干就干，我先试着把整个common项目打了个包，测试下包大小，结果显示在`840KB`左右，如果搞成了，不止达到了抽离公用代码避免所有项目打包的目的，而且在加载性能上还有着极大的优化价值。

不用说，待会肯定需要拆分异步加载，这是后话，暂且抛开。

我单独写了个@core/common项目的打包脚本，`output`如下：

```javascript
output: {
	filename: "public.js",
	publicPath,
	chunkFilename: 'app/public/chunks/[name].[chunkhash:8].js',
	libraryTarget: 'umd',
	library: '$common',
	libraryExport: "default",
}
```
> hive架构之前并不支持打包@core下面的项目，因为是作为公共依赖项目存在的，没有必要独立打包

在entry文件中将`commonStore`导出来，在子项目webpack.config.base.js里配置`externals`，
```
"@common": "$common"
```
当遇到`@common`包的导入时，从`$common`变量上取。
打包，在index.html中引入打包结果`public.js`。
```javascript
import store from '@common/store';
```
执行后，确实取到了store！
> 好了，革命离成功又近了一步，但也别太乐观，我们需要解决的问题还有很多。

目前只是成功导出了store，更麻烦的是还有很多工具类、枚举、容器组件、Hoc组件等需要一一处理。
引用这些玩意儿的方式有很多，比如
```javascript
import Moon from '@moon'; // @moon是别名，指向@core/common/public/moon/index.ts
import Moon from '@common/public/moon';
import $http from '@http'; // @http也是别名，指向@core/common/public/moon/http.ts
import { isIH } from '@common/moon/utils/base';
import { ClientType } from '@common/moon/enums/base';
import { withIconFilter, withPageTable } from '@common/hoc';
import FeedBack from '@common/containers/feedback';
// ...
```
诸如此类的引用遍布在20多个子项目中，这是由于`TypeScript`的别名太好用了，所以有很多引用方式。
webpack单纯的`externals`配置已经解决不了这么复杂的引用场景了。

我当时脑海中有两个解决方案：
- 将@core/common中的entry改为全部挂载在根节点上，修改所有的引用方式为
```javascript
import { A, B, C } from '@common';
```
这个方案要改动非常多的项目代码，20多个项目几乎会改得面目全非，也间接影响小伙伴的开发体验（本来可以按文件路径引入的，现在怎么变成这种莫名其妙的玩法了）。

- 寻找能将以上那些千奇百怪的引入方式正确转换成读取umd包的方法。你还别说悬乎，它真的存在！
webpack真的考虑到了这个场景，直接贴代码：
```javascript
externals: [{
		"moment": "moment"
	},
	{
		"moment-timezone": "moment"
	},
	function (context, request, callback) {
		if (/^echarts$/.test(request) && !isDevelopment) {
			return callback(null, 'echarts');
		}
		if (/^@common\/?.*$/.test(request) && !isDevelopment) {
			return callback(null, request.replace(/@common/, '$common').replace(/\//g, '.'));
		}
		if (/^@moon$/.test(request) && !isDevelopment) {
			return callback(null, "$common.Moon");
		}
		if (/^@http$/.test(request) && !isDevelopment) {
			return callback(null, "$common.utils.http");
		}
		callback();
	}
]
```
externals竟然可以通过函数去匹配，满足条件的就使用外部包，不满足就不使用。

此处另一个黑科技就是将request中的路径中的`/`替换为`.`，将`@common`替换为`$common`。webpack会自动去`$common`上按`.`索引你导出的对象中的属性。

你猜的没错，只需要按照不同的引入方式组织你的@core/common项目中导出的对象结构即可！

> 说起来容易，做起来难，我花了一个小时才将那些不同引入方式整理出来，并一一对照去还原导出对象的结构。

实际entry.tsx结构如下：
```javascript
export default {
  enums,
  utils,
  business,
  containers,
  hoc,
  initFeToolkit,
  store: commonStore,
  Moon: Moon,
  wrapper: {
    authority: AuthorityWrapper,
    errorBoundary: ErrorBoundary,
  },
  public: {
    enums,
    hoc,
    moon: {
      date,
      client,
      role,
      resourceCode,
      format,
      abFeature,
      behavior,
      message: _message,
      base: moonBase,
    }
  }
};
```

此时，我已经实现了将@core/common打包结果public.js从index.html直接引入，让20多个子项目不改动一行代码成功从public.js读取所有common引用，平滑升级成功！

但是页面渲染出来之后，随便点了点便发现了几处问题：其中一个最严重的问题就是`globalFilter`下拉的级联功能没了！好好的怎么就没了呢，我也没动`fe-toolkit`的代码啊！

一般出现这种状况，我第一时间就怀疑`fe-toolkit`的多重分身问题！
因为@core/common也引入了，子项目也引入了，存在两份`fe-toolkit`！
但是之前优化过一次，让不同分身共享同一份`filterStore`，会不会是这里的问题呢？
通过简单的调试代码，发现确实是这个问题 
> 具体原因是：级联的事件订阅发生在`globalFilter.tsx`组件中，存在多份；而事件触发却在`filterStore`中，`reactiveWO`中引入的`fe-toolkit`的`filterStore`是共享`public.js`中的那份，但是事件却是在`reactiveWO`中依赖的`fe-toolkit`中注册的。所以级联变化时，通知到了`public.js`里的事件总线，但是却找不到订阅方，因为订阅方是`reactiveWO`中的。

能怎么办呢？只能将`fe-toolkit`也抽离出来呀！照搬`public.js`的方式，将`fe-toolkit`也抽离成`umd`包。

但是`fe-toolkit`中的`http.ts`打包时的相对路径也要处理一下，否则开发环境与生产环境就兼容不了。也可以做成开发环境依赖npm而生产环境依赖umd js，不过之前有同事一直在抱怨说更新`fe-toolkit`版本太麻烦，那与其如此，不如直接做成umd js依赖，拉一下代码就能使用最新的版本，何乐而不为呢！

在@core/common项目中提供一个initFeToolkit的方法，让子项目调用一下，内部使用fe-toolkit的setAxios方法将common的http实例设置进去，另外顺便注册一下globalFilter的几个查询组件。
> 此前是在子项目入口里注册的查询组件，为什么提取到这里来呢？
> - 省空间，查询组件本来位于@core/components项目，在子项目中注册的话会打包到每一个子项目中去。在这里注册就打到了public.js中，瘦身了35KB左右的大小。
> - 将重复出现的代码抽离，代码更美观，易于控制。
> 


#### 但是，回过头来看此时的`public.js`，还有`840KB`，还得拆呢！
`840KB`意味着你虽然成功的将公共部分抽离了出来，但是仍然没有做到加载性能的优化，因为任何一个子项目都不太可能将@core/common中的工具或组件用全！换句话说就是`840KB`中有一大半是浪费的。

此时又要使用异步加载了，将所有的容器组件打成`chunk`这点无可厚非，完了之后，public.js剩余`430KB`。

试着将`tsconfig.json`中的`target`值改为`esnext`之后，只剩`380KB`。

但是，这大小仍然无法让我感到满足：瘦身是会上瘾的！还有优化的空间！要对Hoc组件下手！

可是找遍全网也没找到异步加载Hoc的资料……得，还得靠自己写。
```javascript
export const withIconFilter = (filterType: FilterType) => {
  return (props: any) => {
    return <LazyLoad
      modules={{
        withIconFilter: () => importLazy(new Promise((resolve, reject) => {
          require.ensure(['../public/hoc/withIconFilter'], require => {
            resolve(require('../public/hoc/withIconFilter'));
          }, error => console.log(error), 'withIconFilter');
        }))
      }}
    >
      {
        ({ withIconFilter }) => React.createElement(withIconFilter(filterType), props)
      }
    </LazyLoad>;
  };
};
```
嘿！成功加载了Hoc，并调用绘制成功了！

没过一分钟，我就发现了bug，为啥Hoc生成的组件总是remount？凭直觉改了下LazyLoad组件的scu，如果isLoaded为true的话就return false了。
嗯，remount的bug暂时消失了。

没过五分钟，我又发现了另一个问题：为啥有些Hoc生成的组件点了没反应？调试了一下代码，发现根本没更新！！！好嘛，还是上一步犯的错，不能不让子组件渲染啊！于是又把scu的判断去掉了。
那怎么解决这个remount的问题呢？理论上children为函数时，React是可以重用子组件的啊，为啥会不断的`unmount`呢？

调试了将近2个小时，才发现原来问题出在下面这句话身上
> 这是个巨坑，一定要记住，不能再踩了，太难受。
```javascript
{
    ({ withIconFilter }) => React.createElement(withIconFilter(filterType), props)
}
```
每次返回的都是通过Hoc新生成的组件产生的React.Element，难怪React会认为每次返回的都是不同的节点了。
重新修改下代码，判断是否已经通过Hoc生成过Component，若已生成则重用，问题解决！
```javascript
export const withIconFilter = (filterType: FilterType) => {
  let Component = null;
  return (props: any) => {
    return <LazyLoad
      modules={{
        withIconFilter: () => importLazy(new Promise((resolve, reject) => {
          require.ensure(['../public/hoc/withIconFilter'], require => {
            resolve(require('../public/hoc/withIconFilter'));
          }, error => console.log(error), 'withIconFilter');
        }))
      }}
    >
      {
        ({ withIconFilter }) => {
          if (Component === null) {
            Component = withIconFilter(filterType);
          }
          return <Component {...props} />;
        }
      }
    </LazyLoad>;
  };
};
```
又优化了一版之后，使用简洁多了，如下：
```javascript
export const withIconFilter = lazyHoc(() => import(
  /* webpackChunkName: 'withIconFilter' */
  '../public/hoc/withIconFilter'
));
```
`lazyHoc`实现如下：
```javascript
const lazyHoc: ILazyHoc = (factory, loading = <></>) => {
  return (...rest: any[]) => {
    const Lazy = React.lazy(async () => {
    const Component = (await factory()).default(...rest);
    return {
      default: props => <Component {...props} />
    };
  });
  return (props: IKeyValueMap) => <React.Suspense fallback={loading}>
      <Lazy {...props} />
    </React.Suspense>;
  };
};

type ILazyHoc = (factory: () => Promise<{
  default: IHoc;
}>, loading?: React.ReactElement) => (...rest: any[]) => React.FunctionComponent<(props: IKeyValueMap) => JSX.Element>;

type IHoc = (...rest: any[]) => React.ComponentClass<any, any>;

export default lazyHoc;
```


拆分将近10个Hoc之后，public.js只剩`283KB`了，好吧，我满足了。
于是抽离公共代码与优化加载性能两个目的便达到了，记录于此，以便不时之需……


噢，忘了说了，经过这一轮优化之后，reactiveWO.js只剩`184KB`，project.js更甚，只剩`20KB`。
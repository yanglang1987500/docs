### 项目初期
打包大小
模块|尺寸
-|:-:
vendor.dll.js|1586KB
reactiveWO|818KB
recurringWO|753KB
dailyMap|606KB
project|712KB

### 2019-08-22 优化
- 分离dll，单独echarts.dll.js
- lodash换成lodash-es，弃用lodash-decorators
- @core开启tree-shaking

打包大小
模块|尺寸
-|:-:
vendor.dll.js|838KB
echarts.dll.js|748KB
reactiveWO|688KB
recurringWO|537KB
dailyMap|268KB
project|472KB

### 2019-08-26 优化
- 抽离Header组件
- 将react-awesome-popover加到vendor.dll中

打包大小
模块|尺寸
-|:-:
vendor.dll.js|878KB
echarts.dll.js|748KB
header|121KB
~quickSearch|101KB
~vendors-quickSearch|160KB
reactiveWO|604KB
recurringWO|370KB
dailyMap|269KB
project|238KB

### 2019-08-28 优化
- 将moment与moment-zone从dll抽离至common-plugin

打包尺寸
模块|尺寸
-|:-:
vendor.dll.js|435KB
echarts.dll.js|748KB

### 2019-09-02 优化
- 将echarts由打包npm版本到vendor.dll改为打包cdn定制功能版本到子项目的plugin中
> 定制版本比npm版本小了**331KB**

打包尺寸

模块|尺寸
-|:-:
vendor.dll.js|435KB
echarts.dll.js|417KB

### 2019-09-06 优化
- 抽离`@core/common`与`fe-toolkit`依赖，打成umd包

打包尺寸

模块|尺寸
-|:-:
vendor.dll.js|435KB
echarts.dll.js|417KB
fe-toolkit.js|67KB
public.js|285KB
~async-chunks+22|560KB
reactiveWO|184KB
recurringWO|105KB
dailyMap|42KB
project|20KB

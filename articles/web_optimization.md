# 前端项目优化解决方案

-----

## 面临问题

目前公司前端项目面临的一个普遍的问题就是页面加载慢，用户操作偶尔不流畅，这无疑降低了系统使用舒适度，从技术层面上看，首先是静态资源体积过大，导致网络传输和解析运行等过程耗时较长；其次是编码方式有较大的优化空间，可以避免渲染次数，进而提高代码执行效率。本方案这次主要聚焦在js文件的优化上，后续将对代码编写方面推出解决方法。

下面选取了两个在IT环境的项目首页加载过程中js的截图(无缓存)：

![图片描述](https://github.com/lanpangzi-zkg/docs/blob/master/articles/images/app-1-slow.jpg)

![图片描述](https://fulu-common-util.oss-cn-hangzhou.aliyuncs.com/wiki_assets/app-2-slow.jpg)

两个系统首页都包含一个超过2M的js文件，网络请求时间很长，其他网络请求阻塞严重，DomContentLoaded都超过了7s，用户看到页面的时间就更长了，因此网站的性能是很差的，目前公司内部还有很多这样的系统，因此亟需一次性能优化。
## 解决方案
#### 1.调整公司组件库引用
目前使用公司组件库的方式为import { ... } from 'fl-pro'，这样使用的问题就是不能实现按需加载，导致组件库所有资源都被打包输出，导致js体积达到1M以上，极大降低了网络传输和js解析速度，现统一调整引入方式如下：
```
import PublicLayout from 'fl-pro/lib/model/PublicLayout'; // 运营测框架index.js文件中的引用需要加model，这里请注意

import FLayout from 'fl-pro/lib/FLayout';
import Table from 'fl-pro/lib/Table';
......
```
* 其他组件类似，都遵循：import 组件名 from 'fl-pro/lib/组件名';

#### 2.三方优化插件
1.) antd-icon-reduce-plugin&antd-icon-reduce-loader
这两款插件是自研的antd图标库体积优化插件，可以减小js体积达数百KB，具体使用配置请参考：
https://www.npmjs.com/package/antd-icon-reduce-plugin 

* 注意：由于公司内部系统使用了fl-pro组件库，因此在webpack配置文件中确保组件库能被antd-icon-reduce-loader处理，例如：

```
{
    test: (filePath) => {
        if (filePath.indexOf('fl-pro') >= 0 && path.extname(filePath) === '.js') {
            return true;
        }
        return false;
    },
    use: ['antd-icon-reduce-loader'],
}
```

2.) antd-dayjs-webpack-plugin
该插件是antd官方推荐的一款减小moment体积的插件，使用简单，首先安装依赖：
```
npm i antd-dayjs-webpack-plugin -D

```
其次，添加webpack插件配置：
```
 plugins: [
    new AntdDayjsWebpackPlugin({
		preset: 'antdv3'
	})  
] 
```
* 注意：请检查日期组件是否有乱码或使用异常的情况，如果有的话，可先不使用这款插件

#### 3.构建输出优化
通过上述优化之后，项目输出的资源体积可能还会比较大，往往都是由于使用了大量的antd组件或者三方组件导致的，因此还需要针对这些情况做一些处理，具体如下：

1.) 提取重复使用的antd组件
```
optimization: {
      splitChunks: {
          chunks: 'all',
          name: true,
          cacheGroups: {
               ......
               antd: {
                  minChunks: 2,
                  priority: 4,
                  test: /node_modules[\\/]antd/,
               },
          },
      },
      runtimeChunk: "single",
},
```

2.) 分离体积较大的三方组件
如果组件支持按需加载的功能，引用方式就调整一下，和公司组件库调整类似，如果不支持，那么使用splitChunks分离出来，形如：
```
optimization: {
      ......
      splitChunks: {
 		......
 		bizchart: {
          		test:  /node_modules[\\/]bizcharts/,
          		name: 'bizchart',
          		priority: 5,
        	},
	}
}
```
 __注意：以上只是一个示例，不一定适合所有项目，因此还需要实际分析才能得到最优结果；__ 

splitChunks相关资料：

https://www.webpackjs.com/plugins/split-chunks-plugin/ 

https://juejin.im/post/5af1677c6fb9a07ab508dabb 

#### 4.其他优化
1.) 图片优化
在一些项目中使用的图片体积较大，可以使用图片压缩工具进行压缩，例如：
https://www.bejson.com/ui/compress_img/ 

2.) cdn
使用一些三方js库的时候可以利用cdn加速访问，如:https://www.bootcdn.cn/， 提供了常用的js库；

## 项目优化展示
#### 1.通行证项目
##### 1.1) js文件输出列表
优化前：
![图片描述](https://fulu-common-util.oss-cn-hangzhou.aliyuncs.com/wiki_assets/tongxingzheng-before.png)

优化后：
![图片描述](https://github.com/lanpangzi-zkg/docs/blob/master/articles/images/tongxingzheng-after.png)

##### 1.2) 体积对比图(最大体积为单个js文件最大的体积)：
![图片描述](https://fulu-common-util.oss-cn-hangzhou.aliyuncs.com/wiki_assets/tongxingzheng-js-duibi.png)

#### 2.福禄研发运维平台(RDMS)

##### 2.1) 体积对比图
![图片描述](https://fulu-common-util.oss-cn-hangzhou.aliyuncs.com/wiki_assets/rdms-js-duibi.png)

##### 2.2) 首页加载性能
以下将通过一组数据来做比较，总共记录了10次网络请求的参数，其中有三个统计指标，分别是Finish，DomContentLoaded，Load，他们分别代表的意思:
Finish：首页渲染完成，包括接口数据的请求，用户可以正常交互；
DomContentLoaded：页面DOM结构加载完成；
Load：页面样式和其他静态资源加载完成；

----------
注：以下统计数据以秒(s)为单位

 __优化前首页记载情况(无缓存)__ 

<table>
  <thead>
    <th>Finish</th>
    <th>DomContentLoaded</th>
    <th>Load</th>
  </thead>
  <tbody>
    <tr>
      <td>12.91</td>
      <td>7.38</td>
      <td>8.41</td>
    </tr>
    <tr>
      <td>14.38</td>
      <td>5.57</td>
      <td>6.18</td>
    </tr>
    <tr>
      <td>10.43</td>
      <td>5.39</td>
      <td>6.19</td>
    </tr>
    <tr>
      <td>10.49</td>
      <td>5.44</td>
      <td>6.12</td>
    </tr>
    <tr>
      <td>10.45</td>
      <td>5.42</td>
      <td>6.17</td>
    </tr>
    <tr>
      <td>10.86</td>
      <td>5.44</td>
      <td>6.08</td>
    </tr>
    <tr>
      <td>11.01</td>
      <td>5.47</td>
      <td>6.12</td>
    </tr>
    <tr>
      <td>11.79</td>
      <td>5.79</td>
      <td>6.80</td>
    </tr>
    <tr>
      <td>10.81</td>
      <td>6.15</td>
      <td>6.72</td>
    </tr>
    <tr>
      <td>11.87</td>
      <td>6.47</td>
      <td>7.29</td>
    </tr>
  </tbody>
</table>


![图片描述](https://fulu-common-util.oss-cn-hangzhou.aliyuncs.com/wiki_assets/rdms-before-argv.png)

 __优化后首页记载情况(无缓存)__ 

<table>
  <thead>
    <th>Finish</th>
    <th>DomContentLoaded</th>
    <th>Load</th>
  </thead>
  <tbody>
    <tr>
      <td>9.27</td>
      <td>3.62</td>
      <td>4.94</td>
    </tr>
    <tr>
      <td>5.23</td>
      <td>2.57</td>
      <td>3.66</td>
    </tr>
    <tr>
      <td>4.94</td>
      <td>2.63</td>
      <td>3.84</td>
    </tr>
    <tr>
      <td>5.82</td>
      <td>2.53</td>
      <td>3.60</td>
    </tr>
    <tr>
      <td>5.22</td>
      <td>2.49</td>
      <td>4.00</td>
    </tr>
    <tr>
      <td>4.12</td>
      <td>2.62</td>
      <td>3.63</td>
    </tr>
    <tr>
      <td>4.13</td>
      <td>2.62</td>
      <td>3.65</td>
    </tr>
    <tr>
      <td>4.14</td>
      <td>2.55</td>
      <td>3.66</td>
    </tr>
    <tr>
      <td>5.91</td>
      <td>3.86</td>
      <td>5.28</td>
    </tr>
    <tr>
      <td>4.32</td>
      <td>2.71</td>
      <td>3.83</td>
    </tr>
  </tbody>
</table>


![图片描述](https://fulu-common-util.oss-cn-hangzhou.aliyuncs.com/wiki_assets/rdms-after-argv.png)

通过上面数据的对比，在三项指标中，性能分别提升了 __54.3%，51.7%，65%__ ，在缓存模式下，性能也可以提高20%以上。
## 最后
在大家整改过程中，需要对打包的js文件进行分析，推荐使用webpack-bundle-analyzer插件，可以清楚的了解每个文件包含的模块，便于优化打包策略。
网站性能评判有很多工具，比如Lighthouse，能自动生成网页性能指标报告，还包含一些调整建议，可以通过这个工具进行评判。
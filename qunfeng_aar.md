

## 两种集成方式

- 依赖源码（调整前）

![old](https://github.com/daBisNewBee/Notes/tree/master/pic/qunfeng_old.png)


- 依赖aar（调整后）

![new](https://github.com/daBisNewBee/Notes/tree/master/pic/qunfeng_new.png)



改进点：

1. 集成方式变更。源码编译调整为中间件aar

1. Flutter项目结构变化。”Flutter Module“调整为”Flutter Application”



## AAR产物比较

![aar](https://github.com/daBisNewBee/Notes/tree/master/pic/qunfeng_aar.png)

## 

## 总结

1. aar方式更为友好，支持开发隔离。

原来开发人员，仅对native的改动，需要配置Flutter环境，Android 环境，每次changes需要编译所有native源码、flutter源码；

现在仅对native的改动，只需要Android环境，不需要配置Flutter环境，仅build native即可。

屏蔽Flutter，达到了隔离的效果，Android开发的同学不要配置flutter环境。



1. aar无法集成依赖至输出怎么办？

”fat-aar”插件支持合并所需sdk(包括flutter项目下的第三方插件)到一个单独的aar，在宿主项目一次依赖即可。



1. Flutter module为何改为Flutter Application?

”fat-aar”插件需要在gradle配置中引入，而Module所需的gradle配置为动态生成，Application默认支持，在静态代码中引入插件更为方便。



## 参考

1. 官方说明:[__将 Flutter module 集成到 Android 项目__](https://flutter.cn/docs/development/add-to-app/android/project-setup)

1. 美团一篇对Flutter介绍比较好的文章：[Flutter原理与实践](https://tech.meituan.com/2018/08/09/waimai-flutter-practice.html)

1. [__Flutter打包AAR插件之fat-aar使用教程__](https://juejin.cn/post/6850037282884452360)

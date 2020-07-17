# 创建项目

当创建一个新安卓项目的时候，有许多基本问题需要考虑，因为这决定者以后的项目走向。目前开发一个 Android 手机上使用的 app 方式不止一种，这给我们带来了很多大的负担和学习成本，其实任何一种技术都有优缺点，如果某种技术占据了绝对优势，那其它技术早就被淘汰了，毕竟再大的学习成本，也有企业对员工的淘汰机制逼着员工快速的转型。之所以有这么多技术类别，还是他们都没有绝对的优势来打败对方。

- 原生开发
- React Native
- Flutter


尽管跨平台开发技术百花齐放，但是由于技术支持的公司决定这这些框架的广泛成度，又加上学习成本的问题，主流的跨平台也就 `React Native` 和 `Flutter`。除了作为技术研究和公司技术选型，作为大众开发人员，我不太偏向于激进的在公司的实际项目中使用新技术。这些新技术势必会给团队的开发进度带来压力，而且新技术的掌握程度很大程度影响到项目的稳定性，更何况这些语言层面的技术，如果不是跨平台的吸引力，很少有人过早的进入项目。这些新技术的使用一般是：

1. 该技术对于原有的技术来说市场反应占据了绝对的优势，考虑后决定使用。
2. 已经成熟的项目，业务稳定了。有多余的人力来研究新技术，实践后觉得带来可观的溢出，逐渐推广迁移到新技术。


对于其他可能的情况，不建议使用新技术，特别是要快速开发应用抢占市场，验证新 idea。


即便是火热的跨平台开发技术，扔要求开发人员对 Native 开发有一定掌握。该技术栈就是主要讲解原生开发中使用到的技术。


## 最低支持的 Android 版本？

如果是做 SDK 开发，可以支持到 4.0 版本之上的，而如果是新 App 开发，都 2020 年了，推荐支持 5.0 以上版本的手机即可。5.0 之上已经达到了 94.1%，具体的数据有多少人是 5.0 以下的手机还在使用的，就很难获得了，何况这些设备很可能是一些电纸书设备。或者是闲置不使用的。之所以不支持这些设备主要是适配带来的问题。基于以下几个问题。

1. 5.0 开始推出了 Material Design 的风格，想要在 5.0 之下使用这些属性开发比较麻烦，而且不够简介。
2. 5.0 之下存在Android打包后.dex文件中方法数64k限制问题，需要引入 `com.android.support:multidex` 支持多 dex 文件问题。这也会增加包体积。为了很快要被迭代掉的设备来适配显然不太划算。连微信，QQ 这种体量的软件都不支持这些设备了，新开发的软件这些设备的用户量还是值得商榷的。


## AndroidX

AndroidX 并不是新东西，而是对 support 库的一个整理，原先的 support 库由于历史的原因需要跟 compileSdkVersion 同时升级，而每次升级总会有一些问题，导致许多开发者不愿意轻易升级。AndroidX 解决了如下的几个问题：


https://www.jianshu.com/p/6ef3924387e7
https://blog.csdn.net/sinyu890807/article/details/97142065



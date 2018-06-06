---
title: React Native第一阶段开发总结
date: 2018-06-05 14:05:14
tags:
---

React Native 作为能同时运行在Android 和 ios上的开发原生app的新技术，现在已经运用在很多应用中，我之前就一直在关注RN的新技术更新，此次刚好跳槽，新东家采用了RN的技术来实现app，于是正式的第一次将RN应用于项目中，在此记录一下学习及开发RN的笔记。

***********

<!--more-->

### 学习资料准备  

作为一名Android开发，其实在学习RN之前，我一直是使用Java开发的，而RN的语法规范使用的是ES6的，对于JS，只存在大学时学过的基础，所以在此推荐两篇学习资料：  

1.《ES6标准入门》 阮一峰出品的语言入门书籍，基本满足了开发RN的前置语法基础；
2.<a href="https://doc.react-china.org/docs/portals.html">《JSX语法讲解》</a>  
RN的布局书写方式采用了JSX的语法糖实现，所以对于在RN中编写APP布局的时候，熟悉JSX就显得尤为重要了，推荐！ 

### Flex布局属性  

在开发项目的过程中，对于布局的属性了解必不可少，而RN的布局属性，结合了css的特性，和原生的Android还是有很大的区别的，在此记录一下：  


#### 属性  

* alignItems 
* alignSelf 
* justifyContent  
* flexDirection  
* flexWrap

| 名称 | 值 |
| - | :-: | -: | 
| alignItems | flex-start , flex-end , center , stretch |
| alignSelf | auto , flex-start , flex-end , center , stretch |
| justifyContent | flex-start , flex-end , center , space-between , space-around |
| flexDirection | row , column |
| flexWrap | wrap , nowrap | 

基本掌握了以上布局的属性，就能完成大部分的RN的界面布局了，下面用一张图讲解一下每个属性的意思，基本采用通俗易懂的方式  

![](https://ww4.sinaimg.cn/mw690/660d0cdfjw1etlhxkrusyj218g0xcwhn.jpg)
![](https://ww3.sinaimg.cn/mw690/660d0cdfjw1etlhxjven9j218g0xcgp4.jpg)
![](https://ww1.sinaimg.cn/mw690/660d0cdfjw1etlhxise4kj218g0xc0vf.jpg)
![](https://ww1.sinaimg.cn/mw690/660d0cdfjw1etlhxjtkfwj218g0xcwhp.jpg)
![](https://ww4.sinaimg.cn/mw690/660d0cdfjw1etlhxl3605j218g0xcwip.jpg)

在RN中，布局的对齐方式是根据主轴，副轴的方向制定的，而flexDirection的属性就是指定主轴的，值为 row ，则主轴为横向的，副轴为纵向，值为column，则主轴为纵向，副轴为横向，默认值为column； 
而ailgnItmes和justifyContent都是表示对齐方式的，justifyContent表示主轴上的对齐方式，ailgnItmes表示副轴上的对齐方式;
ailgnItmes和alignSelf有相同的功能，不过不同点是它是用来让每一个单行的容器居中而不是让整个容器居中。  

### 开发中的小坑总结（在此记录避免以后再犯）

1. 由于在开发之初，对ES6的语法还在一个熟悉阶段，所以经常会犯在{} 中编写判断语句，在此记录，{}中间只能用表达式
2. 如果要跳转的目标页有TextInput，需要在最开始去掉焦点，不然在跳转时会出现卡顿现象
3. 对于IOS输入键盘会遮挡页面，可以用KeyboardAvoidingView来处理，指定属性behavior={__IOS__ ? "padding" : null}基本可以解决  
...一时好多没想起来，待更新。


### 后记 

由于RN尚在Bate版，还存在很多坑得，所以希望选型为RN开发时，需要考虑这些问题。



---
layout: post
title: "Python搭建假接口网站bit-record"
description: "通过Python搭建加接口网站，替代应用开发过程中的“写假接口”过程，提升开发效率，包括接口，用户，评论3部分"
tags: [python]
notes: [python]  
categories: [python]
type: dev
---  
  

[fork](https://github.com/idyllchow/bit-record)

### 背景  

移动应用开发过程中的APP开发阶段，为提升开发效率，服务端的同学通常会根据产品流程图写好所有的"假接口"，移动端根据假接口请求数据调试流程和UI，服务端在开发环境中完成真实接口的逻辑。为简化写假接口这个步骤，用Python搭建这样Web站点，在前端相应的接口后，即可做对应请求。  

### 目的

提升开发效率  

### 实现

参考[廖雪峰博客](http://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000/001432339330096121ae7e38be44570b7fbd0d8faae26f6000)，下面逐一分析说明											
# Spring MVC源码分析

## 看到哪段写得看不下去直接说，我改！

#### Spring MVC还是从 DispatcherServlet 开始

##### DispatcherServlet 继承体系：

![ClassDiagra](media/14886902736101/ClassDiagram1.png)

##### 继承关系图可以清楚， DispatcherServlet 最后也是继承自 HttpServlet 类，只是层层继承，层层封装，添加各种组件，拦截器，处理器，最后成为Spring MVC


-------

#### DispatcherServlet 分为两部分：

* 初始化 init();
* 请求处理 doService(request,response);


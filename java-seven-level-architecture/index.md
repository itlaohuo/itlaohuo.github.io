# 7层项目结构和maven创建项目模板

<!--more-->

### 为什么不是充血模型？

* 现有的业务系统领域模型未拆分。
* 系统调用错综复杂，排查问题难度较大。
* 代码结构错综复杂，排查问题难度较大。
* 充血模型对开发人员技术要求较高。
* 因此充血模型不符合现在的实际情况。

### 为什么是贫血模型？

* 1 对领域模型的依赖不大，上手较快。
* 2 系统调用，通过module隔离开排查问题，方便
* 3 代码结构层级清晰，排查问题方便。
* 4 通过modular的依赖避免了一些技术上的陷阱，如多表操作是不是失效等问题，
* 5 支持脚手架一键生成项目骨架六完善的技术组件支持。
* 6 完善的技术组件支持
* 因此统一使用贫血模型

### 项目分层名词解释

* API层：

> 提供标准的接口、常量和枚举。提供同一对外的接口，可以单独达成价保各外部系统使用。

* Service层：

> API层接口的实现层，DTO转BO

* Manage层：

> 外部系统的衔接层，获取外部接口的返回结果。1、对接DAO：一个manage对应一个DAO，包含对一个表的各种操作，2 对接Integratio：一个manage对应一个integration，获取外部接口的结果。如果把DB，HTTP， rpc点都看成一个接口，management就是为了这些外部系统的出入参做准备的，类似于对外的一个service层。

* DAO层:

> 数据库交互层，与redis，DB等数据库进行交互。DO对象。

* Integration层:

> 外部系统集成层。HTTP，RPC等外部调用从这里开始，获取外围系统的数据。

* WEB层：

> 对外暴露Http服务层

* TASK层:

> 暴露定时任务

* Common层：

> 封装常量、枚举Service，Mange、Integration层调用

* 各个Module内部必须按照业务功能进行划分。如出库、入库、调拨等分成不同的package。

### IDEA对象转化插件

* Codemarker
* easyCode
* GenerateO2O

### 如何通过mvn archetype 命令创建模板

* 1 打包项目 ，在已准备好的项目模板最外层执行打包命令： mvn clean install
* 2 从项目中创建archetype ：在已准备好的项目模板最外层执行命令： mvn archetype:create-from-project
* 3 进入target\generated-sources\archetype,安装archetype到maven仓库中:mvn clean install 或者 mvn clean deploy
* 4 生成archetype-catalog.xml:进入target\generated-sources\archetype,执行 mvn archetype:crawl
* 5 查看archetype-catalog.xml

### 通过maven的acrchetype创建项目

* 1 通过idea创建：new project选择maven，选择 create from archetype ……
* 2 输入命令“mvn archetype:generate” 创建项目骨架,根据提示一步步往下……


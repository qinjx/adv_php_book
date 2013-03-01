# PHPers' Happiness 目录
## 器
### 除了做网页，PHP还能干啥
#### CLI脚本
##### Shell
##### Crontab

#### 桌面GUI
##### wxWidgets（wxPHP）
##### WinBinder
##### GTK

#### 嵌入式语言
##### PostgreSQL plphp

### 编辑器和IDE
#### Vim
#### Sublime Text
#### Eclipse
#### IntelliJ IDEA（PHPStorm）
#### Zend Studio
#### Komodo
#### NetBeans

### 单步调试（需配合编辑器或IDE）
#### Xdebug
#### Zend Debug

### 性能分析(CacheGrind)
#### WinCacheGrind for Windows
#### KCacheGrind for Linux/Mac OS
#### MacCallGrind for Mac OS
#### WebGrind for All

### 单元测试
#### PHPUnit
#### SimpleTest

### 配置管理
#### SVN
#### GIT

### 缺陷分析
#### Zend Code Analyzer from Zend Studio
#### IntelliJ IDEA（PHPStorm）
#### PHP Mess Detector
#### PHP Copy Paste Detector
#### PHP Dead Code Detector

### 代码美化
#### PHP Code Beautifier
#### PHP Code Sniffer

### 持续集成
#### Jekins with PHP Plugin

### 文档生成
#### phpdocumentor
#### APIGen

## 术
### 性能
性能不是最重要的，质量和人的时间更重要
#### 高速存储换低速存储
#### 存储换CPU
#### CPU换网络通信
#### 牺牲动态性和实时性

### 安全
#### Safety
##### 调用者：只有我这里都OK了，我才调用下游
##### 被调者：只有正确的调用，我才继续处理
##### Static和多线程的脏数据隐患
#### Security
##### 数据验证
一切来自客户端的数据都是不安全的（包括$_COOKIE, $_SERVER）
##### 权限检查
只要有入口就会被访问

### 质量
#### 发布前质量保证
##### 测试用例100%覆盖是关键
##### 自动化控制成本
#### 运行时质量监控
##### 再牛逼的程序也要人看着
##### 发布时想好回滚预案

### 可扩展性
#### 分层哲学
##### 松耦合
我是一个类，我不在意。。。
##### 面向接口
我是一个包，推荐xxx跟我合作，但我也可以跟xxx合作

#### 包容心
自造轮子，制定规则的时候，保留使用别人轮子的可能性，例如：

- 定义autoload方法的时候不要占用__autoload()方法，尽量用spl
- autoload加载失败的时候不要抛出异常，让后面可能存在的autoload方法接管

#### 知足常乐
让专业的角色来做专业的事情，反例：

 - 在PHP里实现数据库连接池，主从同步（这是C的活儿）
 - 用PHP Class去生成数据库表（这是SQL的活儿）
 - 用PHP生成大部分HTML元素（这是前端的活儿）

### 可伸缩性
#### 存储分布式
#### 应用防单点
#### 子系统服务化

### 可维护性
#### 不搞潜规则
代码有章可循，明规则
#### 重构
发布产品是贷款，修复Bug是还贷，重构是提前还款，能少还很多利息  
单元测试是重构的保障

### 可部署性
#### 快速迭代，增量升级
#### 横向扩容时，快速部署
#### 开发环境和非开发环境的差异尽量少

## 道
### PHP Web应用架构演化
#### 刀耕火种：include第三方类库
#### 混沌初开：模板引擎将数据和展现分离
#### 男耕女织：MVC
#### 世界大同：自由的N层架构

### 启示录
#### 站在巨人的JB上
#### Don't Repeat Yourself
#### 有限的精力Focus到非人肉不可的事情上去

## 势
### 日常工作更有幸福感
#### 专注于自己最擅长老板最需要的领域
#### 不被迫还连环高利贷

### 职业生涯更有成就感
#### 坚持：每天都进步一点点
#### 积累：像养孩子一样做产品，量变到质变
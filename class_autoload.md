# 细说“PHP类库自动加载”
By *覃健祥（Qin Jianxiang, Author of Lotusphp）2013.1*

* 注1：文档中涉及的国内外框架，我只在2006年-2008年粗浅地看过文档，做过Hello World基准性能测试（CakePHP，Solar，Yii，QeePHP，ThinkPHP），用其中少数几个实际做过项目（CodeIgniter，Zend Framework，Symfony），我对它们的自动加载技术的评述也是以当年版本为基础的。时过境迁，有不少框架的代码发生了翻天覆地的变化，观众朋友请不要将这个文档作为你选择框架的依据。

* 注2：文档中所示的代码段仅用于表达思路，关键代码会用粗体显示并加下划线--markdown版本暂不支持代码加粗。为了文档篇幅不至于太长，以及观众朋友不被过多的代码分散注意力，这些代码片段故意省略了（与要表达的主题思路不太相关的）很多细节。因此，它们一般不能运行，需要详细可运行代码的话，请看文档中提及的开源代码地址，或者自行编写。

## 黑话解释
### 什么是“类库文件”
本文档中所说的类库文件是指PHP library文件，被包含（include/require）的公共文件，他们通常只定义一些class（包括Class, Abstract Class, Interface）或者function。

当然从技术上说，一个文件里如果既有Class定义，也有游离于Class方法体之外的直接执行语句（如echo 'hello world';）也是可以视为类库文件的，只是这种写法不符合好的编码规范。

### 什么是“自动加载”
在PHP代码中，不需要显式地使用文件路径将类库文件包含进来，便可使用该文件中定义的类库，这种技术称作自动加载。

以下几种方式都是自动加载：

* $db = new Db();
* $this->load->library(“Db”); $db = new Db();
* Zend::load(“Zend_Db”); $db = new Zend_Db();
* import(“Zend.Db”); $db = new Zend_Db();

## 为什么要自动加载
在传统的PHP编程实践中，我们一直用include/require来包含类库文件，这种文件包含通常会有如下问题：

### 目录名和文件名变化引起程序代码变化
当类库目录名或者文件名需要更改的时候，所有include了这个文件的php文件也要随着修改，这加大了源代码目录结构重构的负担。

Windows和Unix(Linux/Mac OS)对文件路径大小写、目录分隔符（斜线和反斜线）、不可见字符（如空格）的处理不同，也使得PHP程序员需要花费相当一部分精力来应对文件名和文件路径问题。

### 相对路径的性能问题
我们不会用hard code把类库文件的绝对路径写死在代码里，于是采用相对路径。

一种做法是设置php.ini中的include_path值，然后给include()传入一个相对路径，Zend Framework和Yahoo!就是这样做的，这种方案存在显而易见的性能问题，include_path的值越多，性能损失就越大。包含文件时使用相对路径也会让APC，eAccelerator等Opcode Cache不能有效地缓存他们。(php引擎处理include_path的机制参见http://www.php.net/manual/en/ini.core.php#ini.include-path)

假设在ini文件里include_path=.:/usr/share/pear:/home/admin/taobao，项目里通常要写set_include_path(get_include_path() . “:/var/www/my_proj/zend_framework”)，include(“Zend.php”)时，PHP引擎会依次去./（即当前目录）、/usr/share/pear、/home/admin/taobao、/var/www/my_proj/zend_framework寻找Zend.php，悲剧的是，往往在最后一个目录才会找到（前面的目录都是系统默认的，最后的才是项目中set_include_path设定的），前面的三次尝试导致了性能很差，而且，根据我的测试，file_exists($file)，当$file不存在时，消耗的时间远大于$file存在的情况。include的时候，判断文件是否存在可能也是这个原理（求验证）。

另一种流行的方法是利用"\__FILE__"魔术变量取得应用的根路径，include的时候使用基于“应用根路径”的绝对路径,如include($appRoot . "conf/db.php")，这个方法很好的解决了相对路径带来的性能问题，CakePHP，Symfony，Lotusphp等框架用的这种方案。

### 类库文件间相互依赖的问题
类库文件之间存在依赖，为了保证运行时不出现“类定义找不到”的错误，类库文件会将需要的更基础的类库（如父类和方法体中用到的其它类）包含进来，又为了保证不重复包含，通常要用include_once/require_once，Zend Framework就是这样做的。大量的include_once/require_once也会导致性能问题。

注意，这并不是说 一次include_once比一次include花的时候明显地长，include_once时需要检测这个文件是否已经在included_files列表里了，这个included_files的数据结构应该与字符下标的数组类似（求验证），是一个hash table，查找hash table中某个元素是否存在，时间复杂度是O(1)，所以，include_once比include慢得非常非常不明显，可以忽略不计。

但是，由于有了include_once，框架的开发者和使用者都不再担心重复包含了，于是同一个文件会被include_once多次，这会导致include_once比include执行次数多好几倍，消耗的时间自然也多好几倍了。

当团队里不同水平/不同喜好的成员共同维护一份代码时，这些问题尤其严重：试想，当你接手维护一个项目时，你敢改前辈留下的Class文件名吗？

注：下面讲到的自动加载方法，并不是每一种都能完美地解决这三个问题。

##自动加载的实现方式
从应用开发者（使用开源框架/类库开发上层应用的人，如各类框架的用户）编码的角度，自动加载实现可以分为两大类：

###每次使用类前需显式调用加载方法
即，在使用（new/extends/implements/instanceof/Class::staticMethod）类之前，需要使用框架定义的加载方法显式地将目标类载入进来，传入的参数通常是这个类的名字。例如CodeIgniter的$this->load('Model')，Zend Framework的Zend::load('Zend_Db')，ThinkPHP的import('MyApp.MyAction')。

采用这个方案的框架，对基于它开发的上层应用代码有侵入性，这是一个缺点。如果上层应用开发者使用了ZF开发了应用，想保留Zend _Db，并迁移到CodeIgniter框架上，就不是换一个Autoloader那么简单了，会有大量包含Zend::load()的文件需要编辑并测试。

框架的load()方法接收到调用指令后，先检查一下if class_exists()，如果类不存在，再按一定的规则把参数中的Class名转换成类文件路径（如Zend_Db_Rowset按ZF的规则转换成路径就是Zend/Db/Rowset.php，下划线换成目录分隔符，最后一个下划线后面的字串加php扩展名），然后将这个php文件include进来。类和文件的对应关系不一定是一种固定算法，也可以采用class_name => file_path mapping方式（下面会详细讲到）。

### 每次使用类时直接用，不需写额外代码
即，预先写一些代码，此后，要使用一个类的时候，不用做任何事情，直接使用。就像这样：

	<?php  
	/**
 	* 预先做一堆事情
 	**/
	$obj_1 = new ClassA();
	class ClassB extends ClassC {...}
	class ClassD implements InterfaceE {…}

使用ClassA，ClassC，InterfaceE的时候，不需要调用Zend::load()这样的方法先把它们加载进来。与每次都显式地调用加载方法相比，这个方案最大的优点是没有侵入性，上层用户在写代码的时候不需要关心它要调用的类在哪里，用什么方法加载，直接用他们，就当这些类存在就好了！这种方法有两种实现：

#### 预先将基础类库文件include进来
框架在初始化的时候扫描用户指定的类库目录，然后按照合法的顺序（先父类后子类，先接口后实现）将类库文件自动include进来。

框架之上的应用在使用类的时候就不需要再手动加载了。这个奇葩的方法，我在2008年的时候在lotusphp（当时叫kiwiphp）框架中用过，还没见别人用过 -_-

这个方法存在两个缺点：一是不同开源类库的中定义的Class可能会存在一些功能冲突，即使他们的类名并不相同。二是自动include的类库文件很多（如ZF、Symfony都是10M级别的），每次PHP请求都要解析这么多类，会有性能损失（有Opcode Cache会好一点）。

#### 使用PHP5内置的类自动加载机制
PHP5 引擎内置了一种自动加载类的机制（参见www.php.net/autoload），PHP引擎在执行代码的时候，如果碰到使用了一个没定义的类，不会再像PHP4时代一样抛出一个Fatal Error，而是试图去执行__autoload()方法，__autoload()方法长的就像这样：

	<?php
	function __autoload($className)
	{
		include(“/usr/share/lib/php_classes/$className.php”);
	}

PHP5内置的这个自动加载机制只能加载类（包括抽象类和接口）定义文件，不能加载函数定义文件。

使用spl_autoload_register()方法可以注册多个autoload()函数，这在主流PHP框架中很常见，原因是：

* __autoload()函数是全局唯一的，如果框架占了这个名字，便会导致框架的用户用不了其它的__autoload()方法了，包括用户自定义的和其它类库带的。spl_autoload_register()可以注册多个autoload方法，不存在这个问题。
* __autoload()是一个函数，在实际使用中，开发者势必要赋予它一些变量（例如class path和下文要讲到的class_name => file_path mapping数组）。这就只能靠全局变量（global variable）了，使用全局变量可不是好的编码习惯。spl_autoload_register()可以将一个Class的某个方法注册为autoload函数，如Symfony的spl_autoload_register(array(self::getInstance(), 'autoload'))，和Lotusphp的spl_autoload_register(array($this, "loadClass"));

遗憾的是，有的第三方类库破坏了这种多个autoload共存的机制，它们在不能成功加载的时候抛出一个异常或者打印错误并退出，这就阻止了下一个autoload函数正常运行。

autoload方法接受一个参数：类名，然后根据类名找到类文件路径，再将之include进来，根据类名找文件路径，分为两种实现模式：

##### 类名和路径之间存在明确的运算关系
使用字符串函数（或者再加上数学函数）对类名进行运算，即可得出类定义文件的相对路径（配合应用根路径、第三方类库根路径计算出文件的绝对路径）。Yii是使用这种模式的。如前所述，ZF虽然也是类名和路径之间存在明确运算关系，但不是利用的PHP内置autoload机制。

这种方法仍然不能解决“为什么要使用自动加载”章节提出的“目录名和文件名变化引起程序代码变化”问题。Yii、ThinkPHP、ZF等框架采用这种方法，也有“编码风格统一”方面的考虑，用好的编码实践来规范框架的开发者，也规范框架的使用者，像Java一样，类名和文件名必须匹配。

##### 类名和路径无关，映射关系存在class_name => file_path mapping数组里
即将类名和文件路径的映射关系保存起来，类名作为Key，文件路径作为Value，可以存在数组中，可以序列化后存到本地文件中，也可以存到Opcode Cache中。

这种方法完美地解决了“目录名和文件名变化引起程序代码变化”问题。由于类名和路径无关，采用这种方案的autoloader/框架可以加载任何第三方类库。

## 自动加载案例详解：Lotusphp Autoloader
Lotusphp是我开发的一个PHP框架，它非常简洁（截至2013年1月，不算单元测试，一共才4000多行代码），最初是为我创立的公司快速开发而设计的，那时候它叫kiwiphp，我进Yahoo！ CN后正式开源，就把它改造成门户级框架了，它的松散耦合、分布式存储三剑客（db、cache、search--开发中）、高性能（测试return true的web app，QPS可达原生PHP的60%，超过国内外所有PHP框架）都是源自这个门户级初衷。

Autoloader是lotusphp最广为人用的组件，还记得上面说过松散耦合吗？如果只想用autoloader，除了最基础的Store接口定义和两个Store接口实现，只需要下载Autoloader.php就可以用了，不依赖组件，下载回去且开发者也无须对这个几文件源码做任何个性，要定制和配置它们，调方法或者继承就可以了。

下面我回顾一下Lotusphp的自动加载实现机制：

### 第一阶段：显式调用load()方法
2006年到2007年，lotusphp还没有改名，它的前身叫kiwiphp，kiwiphp用loadCore()，loadExtension()方法来加载框架的核心类和扩展，用loadModel()加载用户自己写的Model。那时候框架的类还很少，一共20K字节，用户的程序也是简单的MVC结构，这种方法满足了当时的需求，雅虎有两个面向消费者服务的项目用到了它。

### 第二阶段：预先将所有基础类库文件include进来
2008年3月，我们在阿里巴巴的创业圣地湖畔花园做一个项目，团队里有两位ySymfony（美国雅虎定制过的Symfony）用户，我们的项目也用了ySymfony，但由于当时所有的第三方框架都只支持单机单库。我们决定把kiwiphp的DB组件拿过来用。我发现Symfony的autoload非常好，只要把kiwiphp的DB目录Copy过去，就能工作了。

Symfony体积庞大（我们用的ySymfony库文件达10M），调试起来非常不方便，一些其它组件的代码稳定性不够，我们便打算把Symfony框架换成Kiwiphp，要在kiwiphp框架下用Symfony的组件可不容易，因为kiwiphp是根据类名计算类文件路径的，而Symfony的类名跟路径没有必然联系。两个框架的组件要和平共处的时候，他们的Autoloader优雅程度如何，高下立判。于是，我决定，借鉴Symfony的Autoloader思路，兼容其它第三方类库。

这中间，我还走了一段弯路，因为当时我还不知道PHP5内置的autoload机制（事实上Symfony那时候已经开始使用spl_autoload_register()了，只是我没深入地看Symfony源码，不知道），于是我自作聪明地发明了这个方法：预先将所有的类库文件包含进来。这种实现的源代码可以在Google Code上找到：http://code.google.com/p/kiwiphp/source/browse/trunk/runtime/kiwi.php?r=2

使用这个方法时，我遇到两个困难：

#### 不同类库之间存在冲突
我们使用了A类库（访问URL 1的时候用到），也使用了B类库（访问URL 2的时候用到），结果由于kiwiphp的策略是预先包含所有的类库，不管访问URL A还是URL 2，都会同时包含A类库和B类库，B的存在总是让A工作不正常，尽管它们的类名并不相同。我们项目组有能力找出冲突并解决了。但我是一个框架开发者，我不能给框架的使用者带来这样的苦恼。

于是，我费心开发了黑白名单功能，允许框架使用者定义一些配置，指定在访问URL 1时哪些文件/目录应该自动包含进来，哪些不应该，以及配置文件没有提及的文件和目录默认是包含还是不包含。类似Apache httpd.conf中的访问权限设置。

客观地说，这个配置规则看起来很强大，但太难用了，还需要使用者去配置，于是甫一推出，便早到同行们的质疑，我在他们的质疑声中发现了PHP5内置的autoload，旋即决定废弃“预先将所有的类库文件包含进来”这一方案。

#### 生产环境性能问题
生产环境，每一次用户请求时都去扫描目录是不可能的，那太慢了。于是我将所有需要包含的类库文件打包成一个大文件，唤作all_in_one.php，访问每个URL都先包含这个all in one文件。include类库文件的IO开销没有了，却增加了一些PHP引擎解析类库代码的开销，好在有Opcode Cache，kiwiphp在默认加载所有组件和类库的情况下，用ab跑Hello World测试，达到了原生PHP的60%，巧合的是，淘宝主搜索现在也采用了这种打包成一个大文件的方案。

60%是一个漂亮的数字，已经领先ZF和Symfony十倍了，但我还想它更进一步，于是我把上层应用的Controller类文件也打包进来了，每个Controller/method组合（对应URL`http://example.com/controller/method/`）都对应一个独立的conteoller_method.php，这个小改进只是减少了一次IO，带来的性能提升非常有限，却给我造成了大麻烦，因为这样的文件体积都大（都是好几MB），数量又多（10个Controller，每个Controller里10个method就能组合100个了），打包好的conteoller_method.php文件们很快把我的Opcode Cache分配的内存吃完了，apache一启动便崩溃。它的实现代码在这里：http://code.google.com/p/kiwiphp/source/browse/branches/release-0.1/runtime/kiwi.php?spec=svn2&r=2 第264行。

### 第三阶段：PHP5 __autoload()
前文说到，黑白名单功能推出后，同行的批评让我发现了PHP5内置了autoload机制，这让我如获至宝，有了它，便可以实现真正的按需加载，即使用户引用的第三方类库有1万个类文件也不怕，用到哪类的时候就加载哪个类。于是，2009年8月，Kiwiphp正式改用PHP5内置的autoload机制：http://code.google.com/p/kiwiphp/source/detail?r=236&path=/trunk/runtime/kiwi.php

kiwiphp虽然可以自动加载其它第三方类库，以一种兼容并包的姿态出现了，但给使用者的自由还不够彻底，一方面Autoloader功能是集成在kiwi.php文件里面的，使用者没有办法只用kiwiphp的autoloader而不用其它诸如MVC的组件；另一方面各个组件对Kiwi核心类和Config有藕断丝连的依赖，或者要调用核心类的方法，或者有的变量是在核心类里初始化的。在采用PHP5内置的autoload方法的同一个月，我决心让kiwiphp的使用者感受到更彻底的自由，不仅kiwiphp可以加载其它框架的类，其它框架也可以随意使用kiwiphp的组件，让各个组件都可以单独被使用者下载使用。这是一次脱胎换骨的升级，我将它改名为lotusphp，自动加载机制没变，但实现细节又经历了几百次修订（SVN revision）。

原理前面已经详细描述过，这里不再赘述，讲讲它的难点吧。

### Lotusphp Autoloader代码实现的难点
扫描源文件中的类库定义

#### 目录遍历
要遍历一个目录及其下的子目录，最容易想到的是递归，然而PHP递归的性能不太好，lotusphp用了一个数组来保存待扫描的目录，如果遇到子目录就把这个子目录push到数组的末尾。Talk is cheap, let me show you the code: 

	<?php
	protected function scanDirs($dirs)
	{
		$i = 0;
		while (isset($dirs[$i]))
		{
			foreach (scandir($dirs[$i]) as $file)
			{
				$currentFile = $dirs[$i] . '/' . $file;
				if (is_dir($currentFile))
				{
					$dirs[] = $currentFile;
				}
			}//end foreach
			$i ++;
		}//end while
	}//end function

完整代码参见：http://code.google.com/p/lotusphp/source/browse/trunk/runtime/Autoloader/Autoloader.php?r=975  第255行

#### 源代码解析
欲知一个PHP文件中定义了哪些类库，一种方法是正规表达式匹配，另外一种是用Tokenizer。

##### 正则匹配
	<?php
	$currentFileContent = trim(file_get_contents($currentFile));
	
	preg_match_all('~^\s*(?:abstract\s+|final\s+)?(?:class|interface)\s+(\w+)(\s+(extends|implements)\s+(\w+))?\s*{~mi', $currentFileContent, $classes);
	foreach ($classes[1] as $key => $class)
	{
		$_classes[$class] = $currentFile;
	}

完整代码参见：http://code.google.com/p/kiwiphp/source/browse/trunk/runtime/kiwi.php?r=2  第112行

正则表达式匹配的优缺点如下：

* 优点：性能较好；代码简单（最难的就是那个RegEx pattern了）。
* 缺点：是某些极端罕见的类定义无法匹配，例如：<?php class /\** class  MyDB{} **/ DB {private $dbh;} ?>。不过，这种写法倒也不是无解，在preg_match之前先用php_strip_whitespace()删除注释，就可以用上面这个正则匹配了。

##### Tokenizer分析
	<?php
	$tokens = token_get_all($src);
	foreach ($tokens as $token)
	{
		list($id, $text) = $token;
		switch ($id)
		{
			case T_STRING:
				if ($found)
				{
					$libNames[strtolower($name)][] = $text;
					$found = false;
				}
				break;
			case T_CLASS:
			case T_INTERFACE:
			case T_FUNCTION:
				$found = true;
				$name = $text;
				break;
		}
	}

完整代码参见：http://code.google.com/p/lotusphp/source/browse/trunk/runtime/Autoloader/Autoloader.php?r=975 第315行

Tokenizer的方法优优缺点正好与正则表达式方法相反：

* 优点：360度无死角地分析PHP文件中定义了哪些类库，无论这些类库定义的写法多少罕见，只要不被PHP引擎报语法错误，就可以被识别出来。
除此之外，Tokenizer方法还可以识别出游离在Class、function定义之外的直接运行的代码，如<?php class DB {…} session_start(); function test() {}?>，这段代码虽然语法层面没有错误，但其中的session_start()是糟糕的编码实践，Tokenizer方法可以将它找出来，而正则表达式是无能为力的，正则没有上下文环境，无论多么精妙的pattern都实现不了这个需求。
* 缺点：编码复杂，需要很小心地处理很多细节，看一下Google Code上的源码即知；另一个缺点是性能很差，幸运的是，Lotusphp提供了性能优化，性能不再是问题。

#### 性能优化
性能优化分两种环境，一是开发环境，开发者在不停地修改调试代码，代码的变更要实时地反映在运行结果中。另一种是非开发环境（包括但不限于：生产环境、预发环境、Daily环境、功能测试环境、性能测试环境），代码只在安装部署的时候发生变化。

Lotusphp autoloader有一个public成员变量$devMode，默认值是true，使用者在非开发环境部署时将其赋值为false，便可自动实现性能优化。

详细代码参见：http://code.google.com/p/lotusphp/source/browse/trunk/runtime/Autoloader/Autoloader.php?r=975 第71行，第103行。

##### 非开发环境性能优化
非开发环境只有在运维人员主动安装部署应用的时候，才需要重新扫描，可将这个扫描结果缓存起来，有安装部署动作发生时，将这个缓存清空。

缓存的介质默认是本地文件，将class_name => file_path mapping和function file list数组序列化后写入本地文件，用到的时候读出来反序列化之。这种缓存机制适用于所有的硬件环境，特别是没有安装Opcode Cache也没有权限修改php.ini的虚拟主机空间，而且这种缓存方式足够快了。

	<?php
	if (true != $this->devMode)
	{
		if ($this->storeHandle instanceof LtStore)
		{
			$this->storeHandle->prefix = 'Lt-Autoloader-' . $this->storeNameSpaceId;
		}
		else
		{
			if (null == $this->storeHandle)
			{
				$this->storeHandle = new LtStoreFile;
				$this->storeHandle->prefix = 'Lt-Autoloader-' . $this->storeNameSpaceId;
				$this->storeHandle->useSerialize = true;
				$this->storeHandle->init();
			}
			else
			{
				trigger_error("You passed a value to autoloader::storeHandle, but it is NOT an instance of LtStore");
			}
		}
	}
	else
	{
		$this->storeHandle = new LtStoreMemory;
	}
	
	if ($storedMap = $this->storeHandle->get("map"))
	{
	   
	}

有独立服务器的用户，可以使用更高效的方式来缓存：Opcode Cache，如APC，需要自己写一个LtStoreApc implements LtStore，然后将之实例化，赋值给autoloader->storeHandle，Lotusphp框架中有个Cache组件，包含APC、eAccelerator、Xcache三种Opcode Cache，可以直接将Lotusphp cache组件用作Autoloader的storeHandle。

##### 开发环境性能优化
开发环境绝大部分时间消耗在这里：遍历并处理目录中的所有子目录及文件。性能优化的原则就是，能缓存的尽量缓存，能不在循环里做的就不要做循环里做。同时，因为在开发环境，使用者做的所有修改应该立刻生效，不需要使用者额外清理缓存。

###### Tokenizer分析结果缓存
Tokenizer分析非常消耗CPU时间， 一个数百行的文件，token_get_all()返回的token多达上千个，这就意味着后面的switch($id)代码段要循环上千次。几十个类文件、几千行类库代码的项目，Tokenizer把所有文件分析一遍，时间消耗是秒级的，在开发环境也能让人感觉到慢了。

在日常的开发工作中，一个项目有很多个类库文件，但我们通常修改一两个类文件就会去刷新网页（或者是运行PHP脚本）查看修改后的效果，如果每次刷新网页时都把所有类库文件的源码分析一遍，其实并不必要，可以把分析结果缓存起来，下次遍历到这个文件的时候，验证一下文件内容有没有改变过，如果没有改变，就直接用上次缓存的分析结果。

如何验证文件内容没有改变过？最初lotusphp是用filemtime()值（即文件最后修改时间）来做比对的，tokenizer分析完之后，记录下该文件的filemtime()，下次遍历的时候获取filemtime()值跟上次记录的对比，若，大于上次记录的filemtime()值，就说明文件内容改变了，反之则认为没有改变。这个方案一直工作地很好，直到2012年12月，我在在网上发现之前雅虎中国的同事报告PHP APC的bug时说，某些情况下，文件内容可能已经完全不同了，但文件的filemtime()却不会更新。的确存在这样的情况，unzip, rsync, rpm, yum/apt-get都有可能导致这种现象发生。为了更可靠地知道文件到底有没有改变过，我在2013年1月换用了“同时比对文件大小和Hash值”的方法，代码如下：

	<?php
	$fileSize = filesize($filePath);
	$fileHash = md5_file($filePath);
	
	$savedFileInfo = $this->persistentStoreHandle->get($filePath);
	if (!isset($savedFileInfo['file_size']) || $savedFileInfo['file_size'] != $fileSize || $savedFileInfo['file_hash'] != $fileHash)
	{
		//token cache not available
	}

详细代码参见：http://code.google.com/p/lotusphp/source/browse/trunk/runtime/Autoloader/Autoloader.php?r=975 第424行

其实，只判断文件Hash值几乎能确定文件是否改变过了，但考虑到Hash也有极小的碰撞概率（比一个人连中10次500万彩票大奖的几率还小吧），增加了文件大小比对，这样就万无一失了。过了几天，我又将md5换成了crc32，因为文件大小相同内容不同的文件，碰撞出相同crc32 checksum的可能性几乎不存在，crc32的运算速度比md5略快一点，详细代码参见：http://code.google.com/p/lotusphp/source/browse/trunk/runtime/Autoloader/Autoloader.php?r=976，第422行。

Tokenizer分析结果缓存是开发环境最重要的性能优化措施，做完它之后，日常开发中每次刷新页面就能控制在1秒以内完成了，可以接受了。

###### 文件扩展名白名单和目录名黑名单
为了跳过一些不需要自动加载的文件和不需要进入的子目录，lotusphp autoloader还设计了扩展名白名单（例如：只允许.php、.inc文件）和目录名黑名单（例如：不要进入.svn、.git目录）功能，并允许使用者配置之。目录名是没办法用白名单穷举的，只能用黑名单告诉autoloader跳过哪些子目录。

还有两个特殊的目录：'.' 和 '..' 分别代表本目录和上级目录，是默认被跳过不扫描的，否则就死循环了。这个不需要使用者配置。

###### 其它优化
此外，还有一些小的优化点，能进一步提升开发环境的性能，但不像tokenizer缓存和黑白名单这么明显，例如：

判断当前文件的扩展名在不在白名单里（或者目录名在不在黑名单里）时，使用isset或者array_key_exists，他们的时间复杂度是O(1)，而不用in_array，in_array的时间复杂度是O(N)。

明确知道数组每个元素的value是字串、数字、数组时，用性能好一点的isset()，不用array_key_exists()。isset是语言结构，array_key_exists()是函数。注意，当数组中存在value为null的情况，则isset不能代替array_key_exists，详情参见PHP手册的array_key_exists章节。

#### 函数文件加载
PHP5内置的自动加载机制只对类有效，函数是没有办法自动按需加载的。lotusphp提供了一个参数给使用者：isLoadFunction，默认值为true。如果此值为true，则自动将扫描中找到的定义了function（不是类中的function）的文件包含进来。代码如下：

	<?php
	protected function loadFunctionFiles()
	{
		if ($this->isLoadFunction && count($this->functionFiles))
		{
			foreach ($this->functionFiles as $functionFile)
			{
				include_once($functionFile);
			}
		}
	}
这里使用了include_once()，include_once与include相比，性能差距可以忽略不计，而且这里每个定义了函数的文件只会被include_once一次，所以不存在性能问题。Lotusphp历史上这里是使用include的，后来为了防止使用者之前手动加载过某个函数文件导致重复加载错误，改为include_once了。改为include_once之后，其实isLoadFunction参数也没有必要存在了，默认将函数文件全部include_once进来即可。

#### 和其它autoload方法共存
使用spl可以注册多个autoload方法，PHP引擎的规则是，如果执行完第一个autoload方法，还是找不到这个类定义，但尝试去执行第二个autoload方法，以此类推。这就要求autoload方法只在能找到类定义文件时将文件包含进来，找不到的时候不要抛出异常不要退出（最好啥都不做）。

	<?php
	protected function loadClass($className)
	{
		if ($filePath = $this->getFilePathByClassName($className))
		{
			include($filePath);
		}
	}

这里为什么不使用include_once 呢，不担心使用者手工包含过类文件导致重复加载吗？如果使用者手工加载了某个类，使用这个类的时候PHP引擎在栈里能找到，就不会触发内置的自动加载机制啦！

#### 测试
Autoloader的protected方法（如分析源码里定义了哪些类库的parseLibNames()方法）测试与平常的字符串处理函数测试类似，传入参数，检测其返回值或者类属性的值或者文件、缓存里的内容，测试完毕，可以将这个TestCase引入的数据消除。

但它的自动加载功能测试非常特殊，因为，上一个TestCase运行完之后，测试过程中include进来的class文件就在那里了，没有办法清除，这会对下一个TestCase的assert()正确性产生影响，如果下一个TestCase还是测试同一个Class，那么class_exists()一定会返回true的，这其实是上个TestCase的遗产，并不能证明这个TestCase测试通过了。所以，这些用于测试自动加载功能的TestCase用到的素材（类文件，函数文件）最好不要包含相同的类名、函数名。

测试的另外一个难点是测试用例如何100%覆盖，这不是Autoloader特有的问题，所有的测试都会遇到这个问题。不过这是另一堂专门的课，以后单独开课细讲。

Lotusphp框架所有组件都会包含三种测试用例：RightWayToUse, WrongWayToUse, PerformanceTuning。在测试自身功能的同时，分别向使用者演示正确的使用方法、错误的使用方法、性能测试和性能调优。

#### 还能优化吗？
能！非开发环境的性能优化还没到头，可以把Autoloader做成PHP扩展（C写的extension），把Autoloader配置（需要扫描的目录、黑白名单、是否自动包含函数文件）写入php.ini，这样做的好处是：

##### 真正对应用无任何侵入
应用层不需要写和Autoloader有关的代码：不用包含Autoloader.php，不用实例化Autoloader Class，不用配置autoloader实例的参数。一台机器只要启用了autoloader扩展，运行在这个机器上的应用就能享受自动加载的好处了。

使用了框架的应用，一般都会用到单一入口模式，和Autoloader有关的代码写在入口文件中，或者框架自己的入口代码内置了，Autoloader对它们的侵入性很小。C扩展版本的Autoloader可以让那些没用单一入口模式的应用也享受到自动加载的好处。

如果多个应用运行在一台机器上，且包含类名相同文件名不同的类，各个应用就要在代码里自行指定需要扫描的目录了，这种情况有办法通过运维部署手段解决，把这些应用分开部署到不同的虚拟机上即可。

##### 性能进一步微幅提升
纯PHP实现的Autoloader只能缓存class_name => file_path mapping，以下这些工作每次HTTP请求（为了简化问题，此处只讨论Web应用，CLI应用对性能优化的需求没有Web那么大）都会被执行：
应用代码中：实例化Autoloader、改变Autoloader成员变量的值
Autoloader.php中：根据autoloadPath计算storeHandle的prefix、判断devMode值以决定使用哪个storeHandle、从storeHandle中取出mapping数组、spl注册

做成PHP扩展后，这些操作只需要在php进程（apache mod_php或者php fast_cgi）启动的时候执行，不需要每次HTTP请求都执行上面这些操作。mapping数组可以直接放在内存里，也不必依赖opcode cache了。

要把Lotusphp Autoloader全部的功能用C实现一遍，代码量还是有一点的，我们用C实现的初衷是“非开发环境”的易部署和高性能，可以考虑最复杂的扫描文件部分还是保留纯PHP代码，用类似spl_autoload_register的机制或者php.ini设置把PHP实现关联到C扩展指针上去，让autoloader C扩展在需要扫描文件的时候（如php进程重启的时候）来调用php实现的扫描方法。

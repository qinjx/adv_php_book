完备的单元测试
===========
## 我对测试的几个基本观点
### 100%覆盖
#### 测试的核心是用例100%覆盖
是指一个被测试方法的各种情况尽可能遍历到

#### 不是指Code Coverage 100%
Code Coverage即自动扫描工具扫出来的测试覆盖率，它只反应了有多少个类多少个方法有测试用例，并不能反应，这个类，已有的测试用例够不够全面

#### 伤其十指不如断其一指
100个方法每个方法一个测试用例，不如一个方法有100个测试用例，

### 自动化运行
#### 100%覆盖的前提是自动化运行
若不能实现自动化测试，人工运行和判读结果的时间成本太高。每天回归测试和持续集成将不具备可行性。没有自动化运行的支持，测试用例设计得越多，测试人员的工作负担越大，反倒是甜蜜的烦恼啦。

#### 服务端自动化测试工具
主要有xUnit系列：

- Junit (for Java)
- PHPUnit and SimpleTest(for PHP)
- OCUnit (for Objective-C)
- NUnit (for .Net)
- DBUnit (for SQL DB)

#### 浏览器端自动化测试工具
主要有:

- Selenium
- Watir
- HTTPUnit
- HTMLUnit
- JSUnit

__注__：上述工具虽然被归为浏览器端测试，但实际上，也可以运行在服务端，如JSUnit可用于服务端Node.js代码的自动化测试

### 变化的业务，不变的接口
#### 业务变化快
我们经常碰到的第一个问题是，业务变化快，担心花大力气编写的测试用例还没跑够一个月就又用不上了。我常听同事苦闷地说：我这个业务变化太快，没法做自动化测试。这个问题可以从两方面缓解一下：

- 找出相对变化不快的：如底层操作（存储、引擎）、成熟的主线业务
- 设计可隔离变化的技术方案，如页面布局可以每月一个新版，但GET、POST字段名不变，这样用HTTPUnit运行的测试用例便可以重复使用了

#### 优先在哪做自动化测试
第二个问题是，我们的组织往往追求多快好省，无论是开发人员还是测试人员，用来写测试用例的时候，可付出的人月数有限，优先在哪做自动化测试呢？

- 变化频繁的：再简单的业务，经常变，一定会出bug，要相信墨菲定律
- 逻辑复杂的

### 单元测试和集成验收测试的关系
#### 集成验收测试是质量的最后屏障
目前业界大公司主要靠测试妹子们人肉测试来保障，有的公司甚至没有专职的测试人员，靠开发人员的自律来保障软件质量。

#### 单元测试帮助提高质量和快速定位问题
单元测试帮助开发人员更好地自测，以向测试人员提交更高质量的软件，有利于降低集成测试的成本。

同样一个bug，在开发人员自测阶段被单元测试发现，比在集成测试阶段发现的代价小很多。

若在集成测试阶段出现bug，单元测试可以帮助快速定位问题，一般来说，测试用例覆盖低的方法或类，出现bug的可能性大一些。

#### 工具可以共享
各种xUnit工具可以用在自动化集成测试中

## 实例展示
### 测试对象
这是一个Google的笔试题，简述如下：
> 设有一整数数组，元素个数为N，求其中N-1个元素相乘的最大乘积。

> 例如：输入数组[2,4,5,3]；则返回60，（对应的N-1个元素是[3,4,5]，它比[2,3,4],[2,3,5],[2,4,5]三种组合的乘积都大）。
> 要求：
> 
> - 时间复杂度尽可能低
> - 不可使用除法

### 象征性测试最常见的场景
不少程序员看到这个题目，第一时间就兴高采烈地随意写了两个测试用例，心想：不就是排除最小值嘛。

![image](https://raw.github.com/qinjx/adv_php_book/master/images/way_of_testing/xzxzzx.png)

这代表了一部分公司的测试现状，表现出的问题是：

- 象征性测试了最常见的场景
- 满足了Code Coverage要求，但用例完备度很差

出现这种情况的原因是：

- 没有测试用例评审
- 没有持续优化和维护测试用例，相比之下，程序员们更愿意花时间优化被测试对象的性能、代码行数，甚至是排版格式

#### 测试代码
	class TestCaseNumUtilV0 extends PHPUnit_Framework_TestCase
	{
	    private $numUtil;
	    public function setUp()
	    {
	        $this->numUtil = new NumUtilV0;
	    }
	
		public function test1()
		{
			$this->assertEquals(120, $this->numUtil->findMaxProd(array(2, 3, 4, 5, 1)));
		}
	
		public function test2()
		{
			$this->assertEquals(720, $this->numUtil->findMaxProd(array(6, 5, 4, 3, 2, 1,0)));
		}
	}

#### 实现代码
	class NumUtilV0
	{
		public function findMaxProd($arr)
		{
	
			$arr_len = count($arr);
			$pick_out_index = null;
	
			//找出最小值的key
			for($i = 0; $i < $arr_len; $i++)
			{
					if (null == $pick_out_index || $arr[$i] < $arr[$pick_out_index])
					{
						$pick_out_index = $i;
					}
			}
	
			/**
			 * 计算除了最小值之外其它N-1个元素的乘积
			 */
			$prod = 1;
			for($i = 0; $i < $arr_len; $i++)
			{
				if ($i != $pick_out_index)
				{
					$prod *= $arr[$i];
				}
			}
			return $prod;
		}
	}

### 考虑特殊情况下的用例
若拿V0版做测试用例评审，很快就会发现：

- 没考虑零和负数的情况
- 没考虑传入参数不合法的情况

像阿里巴巴这样的公司通常会有一个测试用例评审的流程，无论什么业务，都至少会考虑边界条件、主干流、分支流。

下面的V1版测试代码，代表了阿里巴巴早期的测试情况。

#### 测试代码
	class TestCaseNumUtilV1 extends PHPUnit_Framework_TestCase
	{
	    private $numUtil;
	    public function setUp()
	    {
	        $this->numUtil = new NumUtilV0;
	//        $this->numUtil = new NumUtilV1;
	    }
	
		/**
		 * 正常流
		 * @test
		 * @dataProvider  正常流测试数据素材()
		 */
		public function 正常流测试($excepted, $input_array)
		{
			 $this->assertEquals($excepted, $this->numUtil->findMaxProd($input_array));
		}
	
		public function 正常流测试数据素材()
		{
			return array(
				array(0, array(0, 0, 1, 2, 3, 4)),//零的个数大于1
				array(200, array(0, -1, -2, 10, 5, 2)),//零的个数等于1 偶数个负数
				array(0, array(0, -1, 2, 3)),//零的个数等于1 奇数个负数
				array(100, array( -1, -2, -10, -5, 10)),//零的个数小于1 偶数个负数
				array(200, array(-5, -10, -2, 4)),//零的个数小于1 奇数个负数
			);
		}
	
		/**
		 * 异常流
		 * @test
		 * @dataProvider  异常流测试数据素材()
		 * @expectedException PHPUnit_Framework_Error
		 */
		public function 异常流测试($input)
		{
			$this->numUtil->findMaxProd($input);
		}
	
		public function 异常流测试数据素材()
		{
			return array(
				array(NULL),//输入为空
				array(1024),//直接输入一个整数，不是数组
			);
		}
	}

用V1版的测试代码来测试V0版的实现代码，会有5个用例测试失败：

- 正常流有三组数据测试不通过：

	- array(200, array(0, -1, -2, 10, 5, 2)),//零的个数等于1 偶数个负数
	- array(100, array( -1, -2, -10, -5, 10)),//零的个数小于1 偶数个负数
	- array(200, array(-5, -10, -2, 4)),//零的个数小于1 奇数个负数
	
- 异常流的两组数据测试全部不通过

下面修改实现代码，让其通过测试：
	
#### 实现代码
	class NumUtilV1
	{
		public function findMaxProd(array $arr)
	    {
	
	        $arr_len = count($arr);
	
	        /*
	         * 先遍历数组找出零、负数、正数的数量
	         * 只做统计，不排序，不做乘法
	         */
	        $amount_zero = 0;//零的个数
	        $amount_negative = 0;//负数个数
	        $min_positive_index = null;
	        $min_negative_index = null;
	        $max_negative_index = null;
	        $the_only_zero_index = null;
	
	        for($i = 0; $i < $arr_len; $i++)
	        {
	            if (0 > $arr[$i])
	            {
	                $amount_negative += 1;
	                if (null == $min_negative_index || $arr[$i] < $arr[$min_negative_index])
	                {
	                    $min_negative_index = $i;
	                }
	                if (null == $max_negative_index || $arr[$i] > $arr[$max_negative_index])
	                {
	                    $max_negative_index = $i;
	                }
	            }
	            else if (0 == $arr[$i])
	            {
	                $amount_zero += 1;
	                $the_only_zero_index = $i;
	            }
	            else
	            {
	                if (null == $min_positive_index || $arr[$i] < $arr[$min_positive_index])
	                {
	                    $min_positive_index = $i;
	                }
	            }
	        }
	
	        /**
	         * Logical control start
	         */
	        if (1 < $amount_zero)
	        {
	            /*
	             * 0的个数大于1，任意取N-1个元素，其乘积都是0
	             * 故无须再判断正数和负数的个数
	             */
	            return 0;
	        }
	        else if (1 == $amount_zero)
	        {
	            if (1 == $amount_negative % 2)
	            {//奇数个负数
	                /*
	                 * 最大乘积只能是0，无需判断正数个数
	                 */
	                return 0;
	            } else {//偶数个负数
	                /*
	                 * 除0之外的N-1个整数乘积最大
	                 */
	                $pick_out_index = $the_only_zero_index;
	            }
	        }
	        else// if (1 > $amount_zero)
	        {
	            if (1 == $amount_negative % 2)//奇数个负数
	            {
	                /*
	                 * 除【绝对值最小的负数】之外的N-1个整数乘积最大
	                 */
	                $pick_out_index = $max_negative_index;
	            }
	            else//偶数个负数
	            {
	                $pick_out_index = $min_positive_index;
	            }
	        }
	
	        /**
	         * 若需要计算N-1个元素的乘积
	         */
	        $prod = 1;
	        for($i = 0; $i < $arr_len; $i++)
	        {
	            if ($i != $pick_out_index)
	            {
	                $prod *= $arr[$i];
	            }
	        }
	        return $prod;
	    }
	}

### 正常流的测试用例考虑更全面些
在写V1版测试用例时，我们已经考虑了5种正常流的情况，和2种异常流的情况，看起来好像已经很全面了，但我们仍然无法用数字度量测试用例的完备程度达到了多少。

看起来用例好像够多了，但是测试评审的时候大伙儿要疯掉的。因为正常流的测试用例都是扁平化的。

> 扁平化结构不利于人脑处理

想象一下，2万人的公司，如果集团CEO要直管每一个人的话，连开个大会人齐了没有都管不过来。所以CEO不会直接管理每一个基层员工，而是有一个树形的管理架构，由一线经理、总监、事业部总经理、集团副总裁逐级向上汇报。

在做测试用例设计和评审时，我们也需要我们需要一棵树，先画好树的树干、一级树枝、二级树枝。。。然后再逐个树枝详细分析应该有哪些树叶，在面对A树枝时，我们就专心对付A树枝，不必关心B、C、D。

怎样画好这棵树呢，这是有科学理论支持的，它叫做MECE分析法：

#### MECE分析法
> Mutually Exclusive, Collectively Exhaustive

意思是：相互独立，完全穷尽

MECE是麦肯锡思维过程的基本准则，即把一个工作项目分解为若干个更细的工作任务的方法。它主要有两条原则：

- 第一条是完整性，分解工作的过程中不要漏掉某项，要保证完整性
- 第二条是独立性，每项工作之间要独立，每项工作之间不要有交叉重叠

回到这个具体的问题上来，影响乘积的因素有：

- 负数的个数：是奇数还是偶数（没有负数也属于偶数个负数）
- 零的个数：大于1个零（无论怎样取N-1个，乘积都为0）、等于1个零、小于1个零
- 其它：除0和负数外，就只剩下正数了，正数没有特殊乘法属性，暂时只分成有正数和没有正数两种情况吧



#### 测试代码
	class TestCaseNumUtilV2 extends PHPUnit_Framework_TestCase
	{
		private $numUtil;
		public function setUp()
		{
	        $this->numUtil = new NumUtilV1;
	//		$this->numUtil = new NumUtilV2;
		}
	
		/**
		 * 正常流
		 * @test
		 * @dataProvider  正常流测试数据素材()
		 */
		public function 正常流测试($excepted, $input_array)
		{
			$this->assertEquals($excepted, $this->numUtil->findMaxProd($input_array));
		}
	
		public function 正常流测试数据素材()
		{
			return array(
				array(0, array(0, 0, 1, 2, 3, 4)),//零的个数大于1
				array(200,array(0, -1, -2, 10, 5, 2)),//零的个数等于1 偶数个负数 有正数
				array(100,array(0, -1, -2, -10, -5)),//零的个数等于1 偶数个负数 无正数
				array(0,array(0, -1, 2, 3)),//零的个数等于1 奇数个负数 有正数
				array(0,array(0, -1, -2, -3)),//零的个数等于1 奇数个负数 无正数
				array(100,array( -1, -2, -10, -5, 10)),//零的个数小于1 偶数个负数 有正数
				array(-10,array( -1, -2, -1024, -5)),//零的个数小于1 偶数个负数 无正数
				array(200,array(-5, -10, -2, 4)),//零的个数小于1 奇数个负数 有正数
				array(50,array(-5, -10, -2)),//零的个数小于1 奇数个负数 无正数
			);
		}
	
		/**
		 * 异常流
		 * @test
		 * @dataProvider  异常流测试数据素材()
		 * @expectedException PHPUnit_Framework_Error
		 */
		public function 异常流测试($input)
		{
			$this->numUtil->findMaxProd($input);
		}
	
		public function 异常流测试数据素材()
		{
			return array(
				array(NULL),//输入为空
				array(1024),//直接输入一个整数，不是数组
			);
		}
	}
与V1版大部分相同，区别只在：正常流数据测试素材多了4种，达到了9种。用V2版的测试用例，来测试V1版的实现代码，有一组数据测试失败：

	array(-10,array( -1, -2, -1024, -5)),//零的个数小于1 偶数个负数 无正数

这是因为V1版的实现代码，没有考虑正数的数量（正数在乘法里没有特殊性质）。下面修改实现代码，让其通过测试：

#### 实现代码
	class NumUtilV2
	{
		public function findMaxProd(array $arr)
		{
	
			$arr_len = count($arr);
	
			/*
			 * 先遍历数组找出零、负数、正数的数量
			 * 只做统计，不排序，不做乘法
			 */
			$amount_zero = 0;//零的个数
			$amount_negative = 0;//负数个数
			$amount_positive = 0;//正数个数
			$min_positive_index = null;
			$min_negative_index = null;
			$max_negative_index = null;
			$the_only_zero_index = null;
	
			for($i = 0; $i < $arr_len; $i++)
			{
				if (0 > $arr[$i])
				{
					$amount_negative += 1;
					if (null == $min_negative_index || $arr[$i] < $arr[$min_negative_index])
					{
						$min_negative_index = $i;
					}
					if (null == $max_negative_index || $arr[$i] > $arr[$max_negative_index])
					{
						$max_negative_index = $i;
					}
				}
				else if (0 == $arr[$i])
				{
					$amount_zero += 1;
					$the_only_zero_index = $i;
				}
				else
				{
					$amount_positive += 1;
					if (null == $min_positive_index || $arr[$i] < $arr[$min_positive_index])
					{
						$min_positive_index = $i;
					}
				}
			}
	
			/**
			 * Logical control start
			 */
			if (1 < $amount_zero)
			{
				/*
				 * 0的个数大于1，任意取N-1个元素，其乘积都是0
				 * 故无须再判断正数和负数的个数
				 */
				return 0;
			}
			else if (1 == $amount_zero)
			{
				if (1 == $amount_negative % 2)
				{//奇数个负数
					/*
					 * 最大乘积只能是0，无需判断正数个数
					 */
					return 0;
				} else {//偶数个负数
					/*
					 * 除0之外的N-1个整数乘积最大
					 */
					$pick_out_index = $the_only_zero_index;
				}
			}
			else// if (1 > $amount_zero)
			{
				if (1 == $amount_negative % 2)//奇数个负数
				{
					/*
					 * 除【绝对值最小的负数】之外的N-1个整数乘积最大
					 */
					$pick_out_index = $max_negative_index;
				}
				else//偶数个负数
				{
					if (0 < $amount_positive)
					{//存在正数
						/*
						 * 除【绝对值最小的正数】之外的N-1个整数乘积最大
						 */
						$pick_out_index = $min_positive_index;
					}
					else
					{
						/*
						 * 除【绝对值最大的负数】之外的N-1个整数乘积最大
						 * 乘积为负
						 */
						$pick_out_index = $min_negative_index;
					}
				}
			}
	
			/**
			 * 若需要计算N-1个元素的乘积
			 */
			$prod = 1;
			for($i = 0; $i < $arr_len; $i++)
			{
				if ($i != $pick_out_index)
				{
					$prod *= $arr[$i];
				}
			}
			return $prod;
		}
	}

### V3版
#### 测试代码
#### 实现代码

### V4版
#### 测试代码
#### 实现代码

以上代码都在这里：https://github.com/qinjx/lotusphp/tree/master/unittest/unittest_example

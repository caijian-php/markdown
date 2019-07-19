# Laravel 源码流程分析及设计模式




### 前言

---

看下面↓↓↓

使用在一定程度上帮助理解

学习围绕三核心

描述（What）->意义（Why）->实现（How）

研究一个框架或一个类或一段代码

首先得看实现了什么

才能理解其中的奥义

才不会觉得乱



### 入口文件

---
获取框架启动时间
```php
define('LARAVEL_START', microtime(true));
```
加载composer->autoload.php

```php
require __DIR__.'/../vendor/autoload.php';
```

加载框架启动文件

```php
$app = require_once __DIR__.'/../bootstrap/app.php';
```

获取`$app`是laravel启动的关键，也可以说$app是用于启动laravel内核的钥匙🔑。随后就是加载内核，载入服务提供者、门面所映射的实体类，中间件，最后到接收http请求并返回结果。

- 加载http核心类

```php
$kernel = $app->make(Illuminate\Contracts\Http\Kernel::class);
```

- http核心类执行处理请求

```php
$response = $kernel->handle(
    $request = Illuminate\Http\Request::capture()
);
```

- 发送响应

```php
$response->send();
```

- 执行请求生命周期的最终操作(...)

```php
$kernel->terminate($request, $response);
```





### 浅谈Laravel核心—依赖注入容器

---

##### 前言

描述

​	是将组件注入到应用程序中的一种行为。

意义

​	解耦。

实现

​	依赖注入有三种形式

- 构造方法注入

```php
class UserProvider{
    protected $connection;

    public function __construct( Connection $con ){
        $this->connection = $con;
    }
    ...
```

- Setter方法注入

同样，我们也可以使用 Setter 方法注入依赖关系：

```php
class UserProvider{
    protected $connection;
    public function __construct(){
        ...
    }

    public function setConnection( Connection $con ){
        $this->connection = $con;
    }
    ...
```

- 接口注入

```php
interface ConnectionInjector{
    public function injectConnection( Connection $con );
}

class UserProvider implements ConnectionInjector{
    protected $connection;

    public function __construct(){
        ...
    }

    public function injectConnection( Connection $con ){
        $this->connection = $con;
    }
}
```



##### 开始

在框架启动文件中 bootstrap/app.php中实例化的类

```php
$app = require_once __DIR__.'/../bootstrap/app.php';
```

```php
$app = new Illuminate\Foundation\Application(
    $_ENV['APP_BASE_PATH'] ?? dirname(__DIR__)
);
```

#### Illuminate\Foundation\Application

###### 描述

框架启动实体

```php
class Application extends Container implements ApplicationContract, HttpKernelInterface
{}
```

#### extends Container

###### 描述

​	最终的真实的容器实体

###### 意义

​	提供真实并且符合当前需要的容器服务

###### 实现

​	具体实现思想



​	使用了以下成员属性

- protected static $instance 当前全局可用的容器（如果有）
- protected $resolved = [] 已解决的类型数组。
- protected $bindings = [] 容器的绑定。
- protected $methodBindings = [] 容器的方法绑定。
- protected $instances = [] 容器的共享实例。
- protected $aliases = [] 注册类型别名。
- protected $abstractAliases = [] 注册的别名由抽象名称键入。
- protected $extenders = [] 服务的扩展闭包。
- protected $tags = [] 所有注册的标签。
- protected $buildStack = [] 目前正在建造的一堆凝固物stack。
- protected $with = [] 参数覆盖堆栈stack。
- public $contextual = [] 上下文绑定映射。
- protected $reboundCallbacks = [] 所有注册的反弹回调。
- protected $globalResolvingCallbacks = [] 所有全局解决回调。
- protected $globalAfterResolvingCallbacks = [] 解决回调后的所有全局。
- protected $resolvingCallbacks = [] 按类类型的所有解析回调。
- protected $afterResolvingCallbacks = [] 所有按类类别解析后的回调。

```php
use Illuminate\Contracts\Container\Container as ContainerContract;
class Container implements ArrayAccess, ContainerContract
{
    /**
     * The current globally available container (if any).
     *
     * @var static
     */
    protected static $instance;

    /**
     * An array of the types that have been resolved.
     *
     * @var bool[]
     */
    protected $resolved = [];

    /**
     * The container's bindings.
     *
     * @var array[]
     */
    protected $bindings = [];

    /**
     * The container's method bindings.
     *
     * @var \Closure[]
     */
    protected $methodBindings = [];

    /**
     * The container's shared instances.
     *
     * @var object[]
     */
    protected $instances = [];

    /**
     * The registered type aliases.
     *
     * @var string[]
     */
    protected $aliases = [];

    /**
     * The registered aliases keyed by the abstract name.
     *
     * @var array[]
     */
    protected $abstractAliases = [];

    /**
     * The extension closures for services.
     *
     * @var array[]
     */
    protected $extenders = [];

    /**
     * All of the registered tags.
     *
     * @var array[]
     */
    protected $tags = [];

    /**
     * The stack of concretions currently being built.
     *
     * @var array[]
     */
    protected $buildStack = [];

    /**
     * The parameter override stack.
     *
     * @var array[]
     */
    protected $with = [];

    /**
     * The contextual binding map.
     *
     * @var array[]
     */
    public $contextual = [];

    /**
     * All of the registered rebound callbacks.
     *
     * @var array[]
     */
    protected $reboundCallbacks = [];

    /**
     * All of the global resolving callbacks.
     *
     * @var \Closure[]
     */
    protected $globalResolvingCallbacks = [];

    /**
     * All of the global after resolving callbacks.
     *
     * @var \Closure[]
     */
    protected $globalAfterResolvingCallbacks = [];

    /**
     * All of the resolving callbacks by class type.
     *
     * @var array[]
     */
    protected $resolvingCallbacks = [];

    /**
     * All of the after resolving callbacks by class type.
     *
     * @var array[]
     */
    protected $afterResolvingCallbacks = [];
    
    ......
}
```

#### implements ArrayAccess  

###### 描述

php原生接口，提供像访问数组一样访问对象的能力的接口。

###### 意义

使单例实例可以直接通过 array 的方式获取，如果该实例没有，则创建。

###### 实现

```php
ArrayAccess {
/* 方法 */
abstract public offsetExists ( mixed $offset ) : boolean
abstract public offsetGet ( mixed $offset ) : mixed
abstract public offsetSet ( mixed $offset , mixed $value ) : void
abstract public offsetUnset ( mixed $offset ) : void
}
```

#### implements  ContainerContract

###### 描述

Container 扩展的容器接口

###### 意义

​	更好地管理容器内的依赖，更好地提供容器服务

###### 实现

​	该接口定义了以下方法

- public function bound($abstract) 确定是否已绑定给定的抽象类型。
- public function alias($abstract, $alias) 将类型别名为其他名称。
- public function tag($abstracts, $tags) 为给定的绑定分配一组标签。
- public function tagged($tag) 解析给定标记的所有绑定。
- public function bind($abstract, $concrete = null, $shared = false)  在容器中注册绑定。
- public function bindIf($abstract, $concrete = null, $shared = false)  如果尚未注册，请注册绑定。
- public function singleton($abstract, $concrete = null)  在容器中注册共享绑定。
- public function extend($abstract, Closure $closure)  “扩展”容器中的抽象类型。
- public function instance($abstract, $instance)  在容器中注册共享的现有实例。
- public function addContextualBinding($concrete, $abstract, $implementation) 向容器添加上下文绑定。
- public function when($concrete) 定义上下文绑定。
- public function factory($abstract) 获取一个闭包来解决容器中给定的类型。
- public function flush() 刷新所有绑定和已解析实例的容器。
- public function make($abstract, array $parameters = [])  从容器中解析给定的类型。
- public function call($callback, array $parameters = [], $defaultMethod = null)  调用给定的Closure /class @方法并注入其依赖项。
- public function resolved($abstract) 确定是否已解决给定的抽象类型。
- public function resolving($abstract, Closure $callback = null) 注册一个新的解析回调。
- public function afterResolving($abstract, Closure $callback = null)  在解决回调后注册新的。


```php
// namespace Illuminate\Contracts\Container;
interface Container extends ContainerInterface
{
    /**
     * Determine if the given abstract type has been bound.
     *
     * @param  string  $abstract
     * @return bool
     */
    public function bound($abstract);

    /**
     * Alias a type to a different name.
     *
     * @param  string  $abstract
     * @param  string  $alias
     * @return void
     *
     * @throws \LogicException
     */
    public function alias($abstract, $alias);

    /**
     * Assign a set of tags to a given binding.
     *
     * @param  array|string  $abstracts
     * @param  array|mixed   ...$tags
     * @return void
     */
    public function tag($abstracts, $tags);

    /**
     * Resolve all of the bindings for a given tag.
     *
     * @param  string  $tag
     * @return iterable
     */
    public function tagged($tag);

    /**
     * Register a binding with the container.
     *
     * @param  string  $abstract
     * @param  \Closure|string|null  $concrete
     * @param  bool  $shared
     * @return void
     */
    public function bind($abstract, $concrete = null, $shared = false);

    /**
     * Register a binding if it hasn't already been registered.
     *
     * @param  string  $abstract
     * @param  \Closure|string|null  $concrete
     * @param  bool  $shared
     * @return void
     */
    public function bindIf($abstract, $concrete = null, $shared = false);

    /**
     * Register a shared binding in the container.
     *
     * @param  string  $abstract
     * @param  \Closure|string|null  $concrete
     * @return void
     */
    public function singleton($abstract, $concrete = null);

    /**
     * "Extend" an abstract type in the container.
     *
     * @param  string    $abstract
     * @param  \Closure  $closure
     * @return void
     *
     * @throws \InvalidArgumentException
     */
    public function extend($abstract, Closure $closure);

    /**
     * Register an existing instance as shared in the container.
     *
     * @param  string  $abstract
     * @param  mixed   $instance
     * @return mixed
     */
    public function instance($abstract, $instance);

    /**
     * Add a contextual binding to the container.
     *
     * @param  string  $concrete
     * @param  string  $abstract
     * @param  \Closure|string  $implementation
     * @return void
     */
    public function addContextualBinding($concrete, $abstract, $implementation);

    /**
     * Define a contextual binding.
     *
     * @param  string|array  $concrete
     * @return \Illuminate\Contracts\Container\ContextualBindingBuilder
     */
    public function when($concrete);

    /**
     * Get a closure to resolve the given type from the container.
     *
     * @param  string  $abstract
     * @return \Closure
     */
    public function factory($abstract);

    /**
     * Flush the container of all bindings and resolved instances.
     *
     * @return void
     */
    public function flush();

    /**
     * Resolve the given type from the container.
     *
     * @param  string  $abstract
     * @param  array  $parameters
     * @return mixed
     *
     * @throws \Illuminate\Contracts\Container\BindingResolutionException
     */
    public function make($abstract, array $parameters = []);

    /**
     * Call the given Closure / class@method and inject its dependencies.
     *
     * @param  callable|string  $callback
     * @param  array  $parameters
     * @param  string|null  $defaultMethod
     * @return mixed
     */
    public function call($callback, array $parameters = [], $defaultMethod = null);

    /**
     * Determine if the given abstract type has been resolved.
     *
     * @param  string $abstract
     * @return bool
     */
    public function resolved($abstract);

    /**
     * Register a new resolving callback.
     *
     * @param  \Closure|string  $abstract
     * @param  \Closure|null  $callback
     * @return void
     */
    public function resolving($abstract, Closure $callback = null);

    /**
     * Register a new after resolving callback.
     *
     * @param  \Closure|string  $abstract
     * @param  \Closure|null  $callback
     * @return void
     */
    public function afterResolving($abstract, Closure $callback = null);
}
```



####  interface ContainerInterface

###### 描述

​	ContainerInterface 容器接口 该接口为最初始(本质)的容器接口

###### 意义
​	在于实现依赖注入容器

###### 实现

​	其定义了两个方法

- get 通过标识符查找容器的条目并返回它。
- has 如果容器可以返回给定标识符的条目，则返回true。否则返回false。

```php
// namespace Psr\Container;
interface ContainerInterface
{
    /**
     * Finds an entry of the container by its identifier and returns it.
     *
     * @param string $id Identifier of the entry to look for.
     *
     * @throws NotFoundExceptionInterface  No entry was found for **this** identifier.
     * @throws ContainerExceptionInterface Error while retrieving the entry.
     *
     * @return mixed Entry.
     */
    public function get($id);

    /**
     * Returns true if the container can return an entry for the given identifier.
     * Returns false otherwise.
     *
     * `has($id)` returning true does not mean that `get($id)` will not throw an exception.
     * It does however mean that `get($id)` will not throw a `NotFoundExceptionInterface`.
     *
     * @param string $id Identifier of the entry to look for.
     *
     * @return bool
     */
    public function has($id);
}
```







# Laravel中间件详解

###### 前言

主要使用了 

- 系统中间件
- 群组中间件
- 路由中间件

在 Laravel 中，中间件的实现其实是依赖于 `Illuminate\Pipeline\Pipeline` 这个类实现的，我们先来看看触发中间件的代码。

很简单，就是处理后把请求转交给一个闭包就可以继续传递了。

简短的代码实现了耦合了全部中间件



#### larave 管道设计 

Laravel 框架中有一个非常有趣的功能，就是 `HTTP 中间件`，我们在定义路由的时候，通过中间件对访问进行过滤。来自外部的请求首先经过全局中间件，若通过，则会继续穿过层层路由组所设置的中间件，在到达目的路由，当然，目的路由也可能定义了个中间件，通过后，该路由的处理对象（如控制器），得到的就是一个经过过滤的请求了。

##### 流程概览

`Pipeline` 组件实现了一个过滤流程:

> 原始数据 ---> 【前置管道】 ---> 目标处理逻辑 ---> 【后置管道】 ---> 结果数据

```php
// then方法实现了将多个方法的调用集中到了同一个闭包方法$pipeline中
// 最终调用$pipeline方法处理$this->passable并return得到最终结果
// 而这些归根结底是将一切顺势转为了参数
// 而参数就是为了更好地传递而存在
// 正因为参数拿来即用
// 非常适合用于解耦
// 解耦的关键在于如何传递
// 只要有效地实现了传递
// 想实现解耦也就不在话下
// 正如上所说如下所绘
// 闭包的优点之一在于可以使得方法作为一个参数传递
	public function then(Closure $destination)
    {
        $pipeline = array_reduce(
            array_reverse($this->pipes), $this->carry(), $this->prepareDestination($destination)
        );

        return $pipeline($this->passable);
    }
```

```php
// 使用传递的闭包函数$destination处理$passable返回处理结果
// 该处理结果作为一个参数传递到array_reduce第二个参数的callable--$this->carry()中使用
// 而$this->carry()则实现了承接上一个闭包结果并调用本次传递进来的闭包方法的构想
// $this->prepareDestination()方法单纯地实现返回并调用传递的闭包$destination处理$pipeline传递进来的参数
// 首当其冲的方法
	protected function prepareDestination(Closure $destination)
    {
        return function ($passable) use ($destination) {
            return $destination($passable);
        };
    }
```

```php
// 关于array_reduce
// $initital作为$func的第一个参数
// 若将$func视为处理上一个返回结果和下一个参数并返回当前返回结果的参数
// 那么$func可视为第一个返回结果
// 若将上述条件中的`下一个参数`视为一个从callble中处理后的返回结果
// 那么array_reduce将作为处理一系列返回结果的参数
// 以此依次取$arr中的参数(参数这个集合广泛,可以取值为callable)合并调用
// 在pipeline中array_reduce的$func处理的参数皆为closure
function fake_array_reduce($arr, $func, $initial=null)
    $temp = $initial;
    foreach($arr as $v) {
        $temp = $func($temp, $v);
    }
    return $temp;
}
// 拓展PS(与本文无关,建议后阅,以免影响连贯理解):$arr与$initial
```

```php
// 例子取自网路
use Illuminate\Pipeline\Pipeline;
$pipes = [
    function ($poster, $callback) {
        // echo 1;
        $poster += 1;
        return $callback($poster);
    },
    function ($poster, $callback) {
        // echo 2;
        $result = $callback($poster);
        return $result - 1;
    },
    function ($poster, $callback) {
        // echo 3;
        $poster += 2;
        return $callback($poster);
    }
];
// 瀑布流写法
echo (new Pipeline)->send(0)->through($pipes)->then(function ($poster) {
    return $poster;
}); 
// 执行输出为 2
// 处理流程详述如下
// then中传递array_reduce中$func处理的第一个闭包$initial
// 因此第一个闭包处理的结果为0
// 紧接着在array_reduce中第一个参数$arr（待处理集合发射栈）被使用array_reverse颠倒
// 当然这个$arr的使用前身是through($pipes)
// 而through的作用在于将传递进来的参数都视为数组
// array_reverse的颠倒能力导致$pipes提供的参数被使用为后进先出（原array_reduce为先进先出）
// 而array_reduce中$func=$this->carry()中返回的结果是一个待调用的闭包
// 闭包外部引用的参数为上一个闭包的结果
// 而不是普通值传递
// 用表达式描述为 $initital($f3($f2($f1)))
// 则其最终执行的顺序依然是f1->f2->f3->$initial
// $pipe处理$passble和$stack
```

```php
    protected function carry()
    {
        return function ($stack, $pipe) {
            return function ($passable) use ($stack, $pipe) {
                if (is_callable($pipe)) {
                    return $pipe($passable, $stack);
                } elseif (! is_object($pipe)) {
                    [$name, $parameters] = $this->parsePipeString($pipe);
                    $pipe = $this->getContainer()->make($name);

                    $parameters = array_merge([$passable, $stack], $parameters);
                } else {
                    $parameters = [$passable, $stack];
                }
                $response = method_exists($pipe, $this->method)
                                ? $pipe->{$this->method}(...$parameters)
                                : $pipe(...$parameters);
                return $response instanceof Responsable
                            ? $response->toResponse($this->getContainer()->make(Request::class))
                            : $response;
            };
        };
    }
```





#### Interface Pipeline

###### 描述

​	中间件`Pipeline`的接口类，管道，顾名思义，请求的通过需要经过管道

###### 意义

​	定义中间件需要实现的功能

###### 实现

- public function send($traveler)  设置在管道上发送的旅行者对象。
- public function through($stops)  设置管道的停止点。
- public function via($method)  设置方法以停止呼叫。
- public function then(Closure $destination)  使用最终目标回调运行管道。

```php
namespace Illuminate\Contracts\Pipeline;
use Closure;
interface Pipeline
{
    /**
     * Set the traveler object being sent on the pipeline.
     *
     * @param  mixed  $traveler
     * @return $this
     */
    public function send($traveler);

    /**
     * Set the stops of the pipeline.
     *
     * @param  dynamic|array  $stops
     * @return $this
     */
    public function through($stops);

    /**
     * Set the method to call on the stops.
     *
     * @param  string  $method
     * @return $this
     */
    public function via($method);

    /**
     * Run the pipeline with a final destination callback.
     *
     * @param  \Closure  $destination
     * @return mixed
     */
    public function then(Closure $destination);
}
```







#### class Pipeline

---

###### 描述

​	实现中间件具体类

###### 意义

​	解耦

###### 实现

- public function __construct(Container $container = null)  创建一个新的类实例。

- public function send($passable)  设置通过管道发送的对象。
- public function through($pipes)  设置管道数组。
- public function via($method)  设置调用管道的方法。
- public function then(Closure $destination)  使用最终目标回调运行管道。
- public function thenReturn()  运行管道并返回结果。
- protected function prepareDestination(Closure $destination)  获取Closure洋葱的最后一块。
- protected function carry()  获取一个表示应用程序洋葱切片的Closure。
- protected function parsePipeString($pipe)  解析完整的管道字符串以获取名称和参数。
- protected function getContainer()  获取容器实例。









#### App\Http\Kernel

---

###### 描述

​	独立出来的过滤请求的部分

###### 意义

​	对于一个 Web 应用来说，在一个请求真正处理前，我们可能会对请求做各种各样的判断，然后才可以让它继续传递到更深层次中。而如果我们用 `if else` 这样子来，一旦需要判断的条件越来越来，会使得代码更加难以维护，系统间的耦合会增加，而中间件就可以解决这个问题。我们可以把这些判断独立出来做成中间件，可以很方便的过滤请求。

###### 实现

​	责任链模式



#### class Kernel

---

###### 描述

​	配置层  最外层中间件核心

###### 意义

​	用于更容易地配置

###### 实现

- protected $middleware 应用程序的全局HTTP中间件堆栈。这些中间件在您的应用程序的每个请求期间运行。
- protected $middlewareGroups 应用程序的路由中间件组。
- protected $routeMiddleware 应用程序的路由中间件。这些中间件可以分配给组或单独使用。
- protected $middlewarePriority 优先级排序的中间件列表。这迫使非全局中间件始终按给定顺序排列。

```php
namespace App\Http;
use Illuminate\Foundation\Http\Kernel as HttpKernel;
class Kernel extends HttpKernel
{
}
```





#### Illuminate\Foundation\Http\Kernel

---

###### 描述

​	实现层 

###### 意义

​	实现HTTP请求的处理

###### 实现

- public function __construct(Application $app, Router $router)  创建一个新的HTTP内核实例。

将中间件分发到对应属性中，等待使用

- public function handle($request)  处理传入的HTTP请求。

使用 Request 和 Response 

- protected function sendRequestThroughRouter($request)  通过中间件/路由器发送给定的请求。

- public function bootstrap()  引导HTTP请求的应用程序。

- protected function dispatchToRouter()  获取路线调度员回调。

- public function terminate($request, $response)  在任何可终止的中间件上调用terminate方法。

- protected function terminateMiddleware($request, $response)  在任何可终止的中间件上调用terminate方法。

- protected function gatherRouteMiddleware($request)  收集给定请求的路由中间件。

- protected function parseMiddleware($middleware)  解析中间件字符串以获取名称和参数。

- public function hasMiddleware($middleware)  确定内核是否具有给定的中间件。

- public function prependMiddleware($middleware)  如果新的中间件尚不存在，则将其添加到堆栈的开头。

- public function pushMiddleware($middleware)  如果新的中间件尚不存在，则将其添加到堆栈的末尾。

- protected function bootstrappers()  获取应用程序的引导类。

- protected function reportException(Exception $e)  将异常报告给异常处理程序。

- protected function renderException($request, Exception $e)  将异常呈现给响应。

- public function getMiddlewareGroups()  获取应用程序的路由中间件组。

- public function getRouteMiddleware()  获取应用程序的路由中间件。

- public function getApplication()  获取Laravel应用程序实例。



#### interface Kernel

---

###### 描述



###### 意义





###### 实现

public function bootstrap()  引导HTTP请求的应用程序。

public function handle($request)  处理传入的HTTP请求。

public function terminate($request, $response)  执行请求生命周期的任何最终操作。

public function getApplication()  获取Laravel应用程序实例。

```php
interface Kernel
{
    /**
     * Bootstrap the application for HTTP requests.
     *
     * @return void
     */
    public function bootstrap();

    /**
     * Handle an incoming HTTP request.
     *
     * @param  \Symfony\Component\HttpFoundation\Request  $request
     * @return \Symfony\Component\HttpFoundation\Response
     */
    public function handle($request);

    /**
     * Perform any final actions for the request lifecycle.
     *
     * @param  \Symfony\Component\HttpFoundation\Request  $request
     * @param  \Symfony\Component\HttpFoundation\Response  $response
     * @return void
     */
    public function terminate($request, $response);

    /**
     * Get the Laravel application instance.
     *
     * @return \Illuminate\Contracts\Foundation\Application
     */
    public function getApplication();
}
```





#### HttpKernel

---

###### 描述



###### 意义



###### 实现



class Kernel implements KernelContract

















































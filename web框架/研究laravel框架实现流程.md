# Laravel 源码流程分析及设计模式




### 前言

---

看下面↓↓↓

使用在一定程度上帮助理解

学习围绕三核心

描述（What）->意义（Why）->实现（How）

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











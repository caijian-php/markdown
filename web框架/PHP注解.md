## PHP注解

### 前言

**元数据**

​	`所谓元数据就是数据的数据，即元数据是描述数据的。`

​	`就象数据表中的字段一样，每个字段描述了这个字段下的数据的含义。`

​	`元数据可以用于创建文档，跟踪代码中的依赖性，甚至执行基本编译时检查。`

​	`许多元数据工具，如XDoclet，将这些功能添加到核心Java语言中，暂时成为Java编程功能的一部分。`

​	`一般来说，元数据的好处分为三类：文档编制、编译器检查和代码分析。代码级文档最常被引用。`

​	`元数据提供了一种有用的方法来指明方法是否取决于其他方法，它们是否完整，特定类是否必须引用其他类，等等。`



**注解**

​	`注解就是附加在数据/代码上的元数据（metadata）。`

​	`框架可以基于这些元信息为代码提供各种额外功能。`

​	`Java中的注解就是Java源代码的元数据，也就是说注解是用来描述Java源代码的。基本语法就是：@后面跟注解的名称。`



**如何读取注解**

​	`读取注解的内容，需要使用反射技术`

```php
<?php
    
/**
 * @Annotation
 * @Target("CLASS")
 * @funcName 注解内容1
 * @method 注解内容2
 * @method 注解内容2-1
 * 213e
 */
class A
{
    public $a = 1;
}

function explainDocument($class, $annotionName, $all = false)
{
    $class = $class instanceof ReflectionClass ? $class : new \ReflectionClass($class);
    $docComment = $class->getDocComment();
    $annotionName = is_array($annotionName) ? $annotionName : array_slice(func_get_args(), 1);
    preg_match("/@Annotation\s*([^\s]*)/i", $docComment, $matches);
    if (!$matches[1] == '*') throw new \Exception('Not declared as an annotation class');

    foreach ($annotionName as $value) {
        $all ? preg_match("/@{$value}\s*([^\s]*)/i", $docComment, $matches) : preg_match_all("/@{$value}\s*([^\s]*)/i", $docComment, $matches);
        $annotions[$value] = !isset($matches[1]) ? null : $matches[1];
    }

    return $annotions;
}

// 注解的解析
dd(explainDocument(A::class, 'Annotation', 'Target', 'funcname', 'method'));
```






































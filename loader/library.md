先谈一下我对加载一个library需要的逻辑的理解：

1. 首先根据参数解析出类名和子目录
2. 解析出挂载在controller实例上的名称
3. 检查挂载名是否被占用，如果被占用，首先检查类是否存在，不存在的话报错异常退出，然后检查controller实例上挂载的是否是类的实例，如果是说明重复加载不需要再做什么退出加载逻辑即可，否则报错异常退出
4. 遍历library路径，尝试加载类，遍历完还没有找到报错异常退出
5. 尝试加载对应config文件
6. 实例化类并挂载在controller实例上

这套逻辑其实是在加载model的基础上改的，允许了重复请求(同一个lib同一个挂载名)(体现在第三步中)，允许实例化library的时候传参(体现在第六步中)。

如果是这样在加载model的基础上改改就完了，但是CI的实现稍微复杂点，这个复杂主要体现在对框架提供的library的处理上。其实我个人认为CI可以把自带的library独立出来，他只需要提供一个加载library的机制就好了。

既然我不是很认同CI加载library的逻辑，就不贴源码解读了。

我唯一觉得写的不错的是这么一段：

```php
if ($subdir === '')
{
	return $this->_ci_load_library($class.'/'.$class, $params, $object_name);
}
```

这里考虑了类在同名子目录下的情况。如果是复杂的library(需要多个类文件但只暴露一个类出来)的话确实是有这个需求。
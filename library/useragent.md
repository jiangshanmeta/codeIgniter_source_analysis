用户代理类主要是用来处理UA的，顺带处理一下支持的语言、字符集之类的。UA的处理最重要的是对应匹配规则，在这里表现为对应的config文件。对于UA的处理分了四个维度：平台类型、爬虫类型、浏览器类型、移动设备类型，后三个其实是有交叉的。

具体的实现上就是结合正则和config文件匹配，这个library有大段互相雷同的代码。

我觉得写得比较好的是这么一段：

```php
protected function _compile_data()
{
	$this->_set_platform();

	foreach (array('_set_robot', '_set_browser', '_set_mobile') as $function)
	{
		if ($this->$function() === TRUE)
		{
			break;
		}
	}
}
```

至于对字符集和语言的处理，说白了就是对```$_SERVER```中对应属性的分析，相关HTTP协议字段可以作为补充了解一下。

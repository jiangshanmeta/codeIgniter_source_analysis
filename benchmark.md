基准测试类的作用是计算两个标记点的时间差和计算内存占用量。

计算时间差思路很简单，打时间戳就好了。计算内存占用量有系统函数。

```php
class CI_Benchmark {
	// 存放标记点和对应的时间戳
	public $marker = array();

	// 打标记
	public function mark($name)
	{
		$this->marker[$name] = microtime(TRUE);
	}

	// 计算两个标记点时间差
	public function elapsed_time($point1 = '', $point2 = '', $decimals = 4)
	{
		// 不传第一个时间点，默认返回整体运行时间
		if ($point1 === '')
		{
			return '{elapsed_time}';
		}

		if ( ! isset($this->marker[$point1]))
		{
			return '';
		}

		// 没有第二个标记点，第二个标记点默认为当前时间
		if ( ! isset($this->marker[$point2]))
		{
			$this->marker[$point2] = microtime(TRUE);
		}

		// 格式化输出时间
		return number_format($this->marker[$point2] - $this->marker[$point1], $decimals);
	}

	// 返回占用内存
	public function memory_usage()
	{
		return '{memory_usage}';
	}

}
```

注意测试类可能返回两个特殊的字符串```'{elapsed_time}'```和```'{memory_usage}'```，这两个字符串最终会在Output类中被替代处理，得到真是的运行时间和内存消耗。
数组辅助函数提供了几个辅助处理数组的小函数。

## element($item, $array[, $default = NULL])

通过索引```$item```返回```$array```中对应的元素，不存在该索引则返回```$default```。

```php
if ( ! function_exists('element'))
{
	function element($item, array $array, $default = NULL)
	{
		return array_key_exists($item, $array) ? $array[$item] : $default;
	}
}
```

这个函数属于看描述就能随手写出来的那种，不多说了。

## elements($items, $array[, $default = NULL])

通过多个索引```$item```返回```$array```对应的多个节点，如果某个索引没有值则返回```$default```。

```php
if ( ! function_exists('elements'))
{
	// 相当于是上面element方法的加强版
	function elements($items, array $array, $default = NULL)
	{
		$return = array();
		// 提供只需要一个字段时简写方法
		is_array($items) OR $items = array($items);

		foreach ($items as $item)
		{
			$return[$item] = array_key_exists($item, $array) ? $array[$item] : $default;
		}

		return $return;
	}
}
```

依然不觉得有什么特别的


## random_element($array)

随机返回```$array```中的一个元素

```php
if ( ! function_exists('random_element'))
{
	function random_element($array)
	{
		return is_array($array) ? $array[array_rand($array)] : $array;
	}
}
```

PHP提供的```array_rand```方法是返回键，我们需要的是值，对```array_rand```方法包装一层就好了。
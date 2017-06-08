类似于加载器的config功能，加载language也只是一个别名

```php
public function language($files, $lang = '')
{
	// 实际上是调用lang实例的load方法
	get_instance()->lang->load($files, $lang);
	return $this;
}
```
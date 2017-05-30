加载器的加载config功能只是Config的load功能的一个别名。

```php
public function config($file, $use_sections = FALSE, $fail_gracefully = FALSE)
{
	// 从超级对象找到config类的实例，调用其load方法
	return get_instance()->config->load($file, $use_sections, $fail_gracefully);
}
```
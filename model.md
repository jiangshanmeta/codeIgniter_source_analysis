在CI中，model是用来和数据库打交道的PHP类。model的基类CI_Model可以说就是建了一个空的类，唯一能说的就是对魔术方法```__get```的使用。在CI的model中，当要访问的属性不存在时，会尝试直接访问controller实例上的属性，这一功能的实现就是利用了```__get```方法。

```php
class CI_Model {
	public function __construct()
	{
		log_message('info', 'Model Class Initialized');
	}

	// 当访问的属性不存在时，尝试访问controller实例上的对应属性
	public function __get($key)
	{
		return get_instance()->$key;
	}

}
```
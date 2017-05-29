日常开发最经常写的就是controller，我们所写的controller都是```CI_Controller```的子类。

```php
class CI_Controller {
	// 缓存controller实例本身
	private static $instance;

	public function __construct()
	{
		// 把自身挂到```$instance```这个静态属性上
		self::$instance =& $this;

		// 把引导文件实例化的类挂载到controller实例上
		foreach (is_loaded() as $var => $class)
		{
			$this->$var =& load_class($class);
		}

		// 加载加载器类
		$this->load =& load_class('Loader', 'core');
		$this->load->initialize();
		log_message('info', 'Controller Class Initialized');
	}

	// 返回controller单例
	public static function &get_instance()
	{
		return self::$instance;
	}

}
```

controller基类上的静态方法```get_instance```我们自身用得少，用的多的是全局函数```get_instance```:

```php
function &get_instance()
{
	return CI_Controller::get_instance();
}
```

全局函数```get_instance```是对CI_Controller的静态方法```get_instance```的一层包装。
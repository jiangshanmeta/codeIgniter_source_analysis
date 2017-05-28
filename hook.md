Hook是为了用户能在不同处理阶段干预执行流程提供的一种机制。

```php
class CI_Hooks {
	// 是否启用hook，默认不启用
	public $enabled = FALSE;

	// 用户定义的钩子
	public $hooks =	array();

	// 缓存在hook类中使用的其他对象实例
	protected $_objects = array();

	// 是否在处理钩子，防止无限循环
	protected $_in_progress = FALSE;

	public function __construct()
	{
		$CFG =& load_class('Config', 'core');
		log_message('info', 'Hooks Class Initialized');

		// 查看配置中是否启用hook
		if ($CFG->item('enable_hooks') === FALSE)
		{
			return;
		}

		// 加载hook的config文件
		if (file_exists(APPPATH.'config/hooks.php'))
		{
			include(APPPATH.'config/hooks.php');
		}
		if (file_exists(APPPATH.'config/'.ENVIRONMENT.'/hooks.php'))
		{
			include(APPPATH.'config/'.ENVIRONMENT.'/hooks.php');
		}
		if ( ! isset($hook) OR ! is_array($hook))
		{
			return;
		}

		$this->hooks =& $hook;
		$this->enabled = TRUE;
	}


	// 对外暴露的方法，调用某个钩子
	public function call_hook($which = '')
	{
		if ( ! $this->enabled OR ! isset($this->hooks[$which]))
		{
			return FALSE;
		}

		// 允许在一个挂钩点添加多个脚本
		if (is_array($this->hooks[$which]) && ! isset($this->hooks[$which]['function']))
		{
			foreach ($this->hooks[$which] as $val)
			{
				$this->_run_hook($val);
			}
		}
		else
		{
			// 在一个挂钩点只挂了一个钩子
			$this->_run_hook($this->hooks[$which]);
		}

		return TRUE;
	}

	// 处理钩子
	protected function _run_hook($data)
	{
		// 调用实例的某个方法，或者匿名函数
		if (is_callable($data))
		{
			is_array($data)
				? $data[0]->{$data[1]}()
				: $data();

			return TRUE;
		}
		// $data除了匿名函数的情况，都是array
		elseif ( ! is_array($data))
		{
			return FALSE;
		}

		// 防止无限循环
		if ($this->_in_progress === TRUE)
		{
			return;
		}

		// 处理文件路径
		if ( ! isset($data['filepath'], $data['filename']))
		{
			return FALSE;
		}
		$filepath = APPPATH.$data['filepath'].'/'.$data['filename'];
		if ( ! file_exists($filepath))
		{
			return FALSE;
		}


		$class		= empty($data['class']) ? FALSE : $data['class'];
		$function	= empty($data['function']) ? FALSE : $data['function'];
		$params		= isset($data['params']) ? $data['params'] : '';

		if (empty($function))
		{
			return FALSE;
		}

		$this->_in_progress = TRUE;

		// 是否是类的方法
		if ($class !== FALSE)
		{
			// 查看是否有缓存的实例，避免无意义的实例化
			if (isset($this->_objects[$class]))
			{
				if (method_exists($this->_objects[$class], $function))
				{
					$this->_objects[$class]->$function($params);
				}
				else
				{
					return $this->_in_progress = FALSE;
				}
			}
			else
			{
				class_exists($class, FALSE) OR require_once($filepath);

				if ( ! class_exists($class, FALSE) OR ! method_exists($class, $function))
				{
					return $this->_in_progress = FALSE;
				}

				// 缓存实例
				$this->_objects[$class] = new $class();

				// 执行方法
				$this->_objects[$class]->$function($params);
			}
		}
		else
		{
			// 面向过程的函数
			function_exists($function) OR require_once($filepath);

			if ( ! function_exists($function))
			{
				return $this->_in_progress = FALSE;
			}

			$function($params);
		}

		$this->_in_progress = FALSE;
		return TRUE;
	}

}

```

真正调用钩子方法的是```_run_hook```，它支持了多种钩子类型：匿名函数、实例方法、面向过程的方法。对于实例方法还做了对于实例的缓存，避免多余的实例化操作。
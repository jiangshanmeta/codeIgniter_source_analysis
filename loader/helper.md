helper是一堆面向过程的小函数，加载helper不像加载model或者library需要实例化一个类，更不需要挂载到controller实例上，只需要把相应的文件include进来就好了。这里唯一需要注意的是需要记录下加载了的helper，一方面是为了避免重复加载，更重要的原因是在php中声明重名函数会报错。

```php
public function helper($helpers = array())
{
	is_array($helpers) OR $helpers = array($helpers);
	foreach ($helpers as &$helper)
	{
		// 处理路径和规格化helper名，helper都以*_helper的形式命名
		$filename = basename($helper);
		$filepath = ($filename === $helper) ? '' : substr($helper, 0, strlen($helper) - strlen($filename));
		$filename = strtolower(preg_replace('#(_helper)?(\.php)?$#i', '', $filename)).'_helper';
		$helper   = $filepath.$filename;

		// 已加载的不再重复加载
		if (isset($this->_ci_helpers[$helper]))
		{
			continue;
		}

		// 查找是否有用户扩展的helper
		$ext_helper = config_item('subclass_prefix').$filename;
		$ext_loaded = FALSE;
		foreach ($this->_ci_helper_paths as $path)
		{
			if (file_exists($path.'helpers/'.$ext_helper.'.php'))
			{
				include_once($path.'helpers/'.$ext_helper.'.php');
				$ext_loaded = TRUE;
			}
		}
		// 有用户扩展的helper，加载对应的框架提供的helper
		if ($ext_loaded === TRUE)
		{
			$base_helper = BASEPATH.'helpers/'.$helper.'.php';
			if ( ! file_exists($base_helper))
			{
				show_error('Unable to load the requested file: helpers/'.$helper.'.php');
			}

			include_once($base_helper);
			$this->_ci_helpers[$helper] = TRUE;
			log_message('info', 'Helper loaded: '.$helper);
			continue;
		}

		// 没有用户扩展的helper
		foreach ($this->_ci_helper_paths as $path)
		{
			if (file_exists($path.'helpers/'.$helper.'.php'))
			{
				include_once($path.'helpers/'.$helper.'.php');
				// 标记这个helper为已加载
				$this->_ci_helpers[$helper] = TRUE;
				log_message('info', 'Helper loaded: '.$helper);
				break;
			}
		}

		
		if ( ! isset($this->_ci_helpers[$helper]))
		{
			show_error('Unable to load the requested file: helpers/'.$helper.'.php');
		}
	}
	// 支持链式调用
	return $this;
}
```

代码本身没有太多难以理解的，但是有个地方我感觉非常不适：加载helper以一组helper为单位，而加载model、library是以一个item为单位，这种不统一我感觉有点不爽，但这仅仅是代码组织上的问题。
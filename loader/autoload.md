所谓自动加载，它并没有提供什么新的功能，只是结合对应的config文件，调用了已经实现好的的加载model、加载helper等功能。


```php
protected function _ci_autoloader()
{
	// 加载对应的autoload的config文件
	if (file_exists(APPPATH.'config/autoload.php'))
	{
		include(APPPATH.'config/autoload.php');
	}
	if (file_exists(APPPATH.'config/'.ENVIRONMENT.'/autoload.php'))
	{
		include(APPPATH.'config/'.ENVIRONMENT.'/autoload.php');
	}
	if ( ! isset($autoload))
	{
		return;
	}

	// 添加路径
	if (isset($autoload['packages']))
	{
		foreach ($autoload['packages'] as $package_path)
		{
			$this->add_package_path($package_path);
		}
	}

	// 加载config
	if (count($autoload['config']) > 0)
	{
		foreach ($autoload['config'] as $val)
		{
			$this->config($val);
		}
	}

	// 自动加载config和language
	foreach (array('helper', 'language') as $type)
	{
		if (isset($autoload[$type]) && count($autoload[$type]) > 0)
		{
			$this->$type($autoload[$type]);
		}
	}

	// 自动加载drivers
	if (isset($autoload['drivers']))
	{
		$this->driver($autoload['drivers']);
	}

	// 加载library
	if (isset($autoload['libraries']) && count($autoload['libraries']) > 0)
	{
		if (in_array('database', $autoload['libraries']))
		{
			$this->database();
			$autoload['libraries'] = array_diff($autoload['libraries'], array('database'));
		}
		$this->library($autoload['libraries']);
	}

	// 自动加载model
	if (isset($autoload['model']))
	{
		$this->model($autoload['model']);
	}
}
```
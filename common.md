CI定义了一些面向过程的全局函数，这些函数是运行框架必须依赖的。

## is_php($version)

这个函数是用来判断当前PHP版本是否大于某个特定版本的

```php
function is_php($version)
{
	static $_is_php;
	$version = (string) $version;

	if ( ! isset($_is_php[$version]))
	{
		$_is_php[$version] = version_compare(PHP_VERSION, $version, '>=');
	}

	return $_is_php[$version];
}
```

可以看到，这个函数本身是对```version_compare```的封装，但是利用静态属性```$_is_php```对判断结果进行缓存。利用静态属性对结果进行缓存不仅应用在面向过程的函数中，还可以应用在类中。


## get_config

看到这个函数你可能会想到Config类，你可能会问既然有了Config类，为啥需要这个方法。事情是这样的，要config类工作，我们需要考虑加载可能出现的子类，因而需要获得```subclass_prefix```这个config项目，于是我们陷入了死循环。我考虑的一个解决方案是在```CI_Config```这个基类上添加静态方法，这样不需要实例化，不需要考虑subclass_prefix了。我的这个方案相当于是把```get_config```这个全局方法封装到了Config类上。


```php
function &get_config(Array $replace = array())
{
	// 缓存结果
	static $config;

	// 没有缓存的结果才去加载对应的config文件
	if (empty($config))
	{
		$file_path = APPPATH.'config/config.php';
		$found = FALSE;
		if (file_exists($file_path))
		{
			$found = TRUE;
			require($file_path);
		}

		// 根据环境加载相应的config
		if (file_exists($file_path = APPPATH.'config/'.ENVIRONMENT.'/config.php'))
		{
			require($file_path);
		}
		elseif ( ! $found)
		{
			set_status_header(503);
			echo 'The configuration file does not exist.';
			exit(3);
		}

		if ( ! isset($config) OR ! is_array($config))
		{
			set_status_header(503);
			echo 'Your config file does not appear to be formatted correctly.';
			exit(3);
		}
	}

	// 替代旧的config，虽然很少用到
	foreach ($replace as $key => $val)
	{
		$config[$key] = $val;
	}

	return $config;
}
```

这个函数的主要作用是加载config数据，特殊情况下可能会更新这个config数组


## config_item

```php
function config_item($item)
{
	static $_config;

	if (empty($_config))
	{
		// 引用不能直接赋给静态属性，用一个array绕过这个问题
		$_config[0] =& get_config();
	}

	return isset($_config[0][$item]) ? $_config[0][$item] : NULL;
}
```

这个函数是用来获取某个特定的config项目的，依赖于```get_config```方法。


## load_class

这个函数是用来加载CI生命周期中核心类的实例的。

```php
function &load_class($class, $directory = 'libraries', $param = NULL)
{
	// 缓存实例化的类
	static $_classes = array();

	// 如果已实例化，返回已经实例化的类
	if (isset($_classes[$class]))
	{
		return $_classes[$class];
	}

	$name = FALSE;

	// 查找是否存在框架定义的类
	foreach (array(APPPATH, BASEPATH) as $path)
	{
		if (file_exists($path.$directory.'/'.$class.'.php'))
		{
			$name = 'CI_'.$class;

			if (class_exists($name, FALSE) === FALSE)
			{
				require_once($path.$directory.'/'.$class.'.php');
			}

			break;
		}
	}

	// 查找是否有开发者扩展的类
	if (file_exists(APPPATH.$directory.'/'.config_item('subclass_prefix').$class.'.php'))
	{
		$name = config_item('subclass_prefix').$class;

		if (class_exists($name, FALSE) === FALSE)
		{
			require_once(APPPATH.$directory.'/'.$name.'.php');
		}
	}

	if ($name === FALSE)
	{
		set_status_header(503);
		echo 'Unable to locate the specified class: '.$class.'.php';
		exit(5);
	}

	// 记录该类已被加载
	is_loaded($class);

	// 实例化
	$_classes[$class] = isset($param)
		? new $name($param)
		: new $name();

	// 返回实例化的类
	return $_classes[$class];
}

function &is_loaded($class = '')
{
	static $_is_loaded = array();

	if ($class !== '')
	{
		$_is_loaded[strtolower($class)] = $class;
	}

	return $_is_loaded;
}
```
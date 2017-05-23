加载一个model主要做了以下几件事：

* 找寻并加载存放对应model类的文件
* 实例化model
* 把实例化的类挂在到controller实例上


```php
public function model($model, $name = '', $db_conn = FALSE)
{
	if (empty($model))
	{
		return $this;
	}
	elseif (is_array($model))
	{
		foreach ($model as $key => $value)
		{
			is_int($key) ? $this->model($value, '', $db_conn) : $this->model($key, $value, $db_conn);
		}

		return $this;
	}

	// 有可能在子目录中
	$path = '';
	if (($last_slash = strrpos($model, '/')) !== FALSE)
	{
		$path = substr($model, 0, ++$last_slash);
		$model = substr($model, $last_slash);
	}

	// 默认在controller实例上的挂载名为model名
	if (empty($name))
	{
		$name = $model;
	}

	// 预防重名
	if (in_array($name, $this->_ci_models, TRUE))
	{
		return $this;
	}
	$CI =& get_instance();
	if (isset($CI->$name))
	{
		throw new RuntimeException('The model name you are loading is the name of a resource that is already being used: '.$name);
	}

	if ($db_conn !== FALSE && ! class_exists('CI_DB', FALSE))
	{
		if ($db_conn === TRUE)
		{
			$db_conn = '';
		}

		$this->database($db_conn, FALSE, TRUE);
	}

	// 加载基类，包括框架提供的```CI_Model```和自行扩展的model基类
	if ( ! class_exists('CI_Model', FALSE))
	{
		$app_path = APPPATH.'core'.DIRECTORY_SEPARATOR;
		if (file_exists($app_path.'Model.php'))
		{
			require_once($app_path.'Model.php');
			if ( ! class_exists('CI_Model', FALSE))
			{
				throw new RuntimeException($app_path."Model.php exists, but doesn't declare class CI_Model");
			}
		}
		elseif ( ! class_exists('CI_Model', FALSE))
		{
			require_once(BASEPATH.'core'.DIRECTORY_SEPARATOR.'Model.php');
		}

		// 加载自行扩展的model基类
		$class = config_item('subclass_prefix').'Model';
		if (file_exists($app_path.$class.'.php'))
		{
			require_once($app_path.$class.'.php');
			if ( ! class_exists($class, FALSE))
			{
				throw new RuntimeException($app_path.$class.".php exists, but doesn't declare class ".$class);
			}
		}
	}

	// 寻找并加载model所在文件
	$model = ucfirst($model);
	if ( ! class_exists($model, FALSE))
	{
		foreach ($this->_ci_model_paths as $mod_path)
		{
			if ( ! file_exists($mod_path.'models/'.$path.$model.'.php'))
			{
				continue;
			}

			require_once($mod_path.'models/'.$path.$model.'.php');
			if ( ! class_exists($model, FALSE))
			{
				throw new RuntimeException($mod_path."models/".$path.$model.".php exists, but doesn't declare class ".$model);
			}

			break;
		}
		// 可能文件存在但对应的类不存在
		if ( ! class_exists($model, FALSE))
		{
			throw new RuntimeException('Unable to locate the model you have specified: '.$model);
		}
	}
	// 要保证加载的类是```CI_Model```的子类
	elseif ( ! is_subclass_of($model, 'CI_Model'))
	{
		throw new RuntimeException("Class ".$model." already exists and doesn't extend CI_Model");
	}

	$this->_ci_models[] = $name;
	// 实例化model，并挂载到controller实例上
	$CI->$name = new $model();
	// 支持链式调用
	return $this;
}
```

实现核心功能的代码其实并不多，大多数代码都是在容错。



我个人觉得写的非常好的一段是：

```php
if (empty($model))
{
	return $this;
}
elseif (is_array($model))
{
	foreach ($model as $key => $value)
	{
		is_int($key) ? $this->model($value, '', $db_conn) : $this->model($key, $value, $db_conn);
	}

	return $this;
}
```

这一段代码支持了array传参，支持了一次加载多个model。这种API设计方式其实非常常见，我在前端的相关实现也是用到了类似的思路。
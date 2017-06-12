配置类的作用是加载、设定和获取配置项目。配置项目最终都存放在```$config```这个数组中。

加载配置项：

```php

// $file 配置文件名
// $use_sections 决定加载的config数组如何合并到config实例的$config数组中
// $fail_gracefully 出错时报错级别

public function load($file = '', $use_sections = FALSE, $fail_gracefully = FALSE)
{
	$file = ($file === '') ? 'config' : str_replace('.php', '', $file);
	$loaded = FALSE;

	// 遍历路径寻找配置文件
	foreach ($this->_config_paths as $path)
	{
		// 允许根据环境设定配置项
		foreach (array($file, ENVIRONMENT.DIRECTORY_SEPARATOR.$file) as $location)
		{
			$file_path = $path.'config/'.$location.'.php';
			if (in_array($file_path, $this->is_loaded, TRUE))
			{
				return TRUE;
			}

			if ( ! file_exists($file_path))
			{
				continue;
			}

			include($file_path);

			if ( ! isset($config) OR ! is_array($config))
			{
				if ($fail_gracefully === TRUE)
				{
					return FALSE;
				}

				show_error('Your '.$file_path.' file does not appear to contain a valid configuration array.');
			}

			
			if ($use_sections === TRUE)
			{
				// 加载的配置项作为一个整体
				$this->config[$file] = isset($this->config[$file])
					? array_merge($this->config[$file], $config)
					: $config;
			}
			else
			{
				// 完全合并配置项
				$this->config = array_merge($this->config, $config);
			}

			// 标记已经加载过的文件
			$this->is_loaded[] = $file_path;
			$config = NULL;
			$loaded = TRUE;
			log_message('debug', 'Config file loaded: '.$file_path);
		}
	}

	if ($loaded === TRUE)
	{
		return TRUE;
	}
	elseif ($fail_gracefully === TRUE)
	{
		return FALSE;
	}

	show_error('The configuration file '.$file.'.php does not exist.');
}
```

代码本身不复杂，我个人认为设计比较好的一点是```$use_sections```这个参数，通过这个参数，我们可以设定配置项的存储结构，当项目本身比较小的时候，直接合并到公有属性```$config```上，当项目较大时，为了防止重名，按照文件名称组织到```$config```上。

```php
public function item($item, $index = '')
{
	if ($index == '')
	{
		return isset($this->config[$item]) ? $this->config[$item] : NULL;
	}

	return isset($this->config[$index], $this->config[$index][$item]) ? $this->config[$index][$item] : NULL;
}
```

上面是获取配置项对应的代码，这个实现考虑到了```load```方法中两种不同的配置项组织形式。

最后一个是设置配置项：

```php
public function set_item($item, $value)
{
	$this->config[$item] = $value;
}
```

在config类中还有几个关于uri的方法，我一直很好奇为什么不放在uri类中。
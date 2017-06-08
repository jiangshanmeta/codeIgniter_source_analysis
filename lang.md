语言类用来处理不同语言的配置文件，它使用公有属性```$language```存放所有的语言配置，使用```$is_loaded```标记已经加载的语言配置文件。语言类有两个公有方法```load```和```line```，前者用于加载某个特定的语言配置文件，后者用来获得特定的语言配置项。

```php
// $langfile 语言文件名
// $idiom 语言类型
// $alt_path 别名路径

public function load($langfile, $idiom = '', $return = FALSE, $add_suffix = TRUE, $alt_path = '')
{
	// 支持一次加载一组语言配置文件
	if (is_array($langfile))
	{
		foreach ($langfile as $value)
		{
			$this->load($value, $idiom, $return, $add_suffix, $alt_path);
		}

		return;
	}

	// 规格化语言配置文件名
	$langfile = str_replace('.php', '', $langfile);
	if ($add_suffix === TRUE)
	{
		$langfile = preg_replace('/_lang$/', '', $langfile).'_lang';
	}
	$langfile .= '.php';

	// 规格化语言类型
	if (empty($idiom) OR ! preg_match('/^[a-z_-]+$/i', $idiom))
	{
		$config =& get_config();
		$idiom = empty($config['language']) ? 'english' : $config['language'];
	}

	// 已经加载且不需要返回结果
	if ($return === FALSE && isset($this->is_loaded[$langfile]) && $this->is_loaded[$langfile] === $idiom)
	{
		return;
	}

	// 加载语言配置文件
	$basepath = BASEPATH.'language/'.$idiom.'/'.$langfile;
	if (($found = file_exists($basepath)) === TRUE)
	{
		include($basepath);
	}
	if ($alt_path !== '')
	{
		$alt_path .= 'language/'.$idiom.'/'.$langfile;
		if (file_exists($alt_path))
		{
			include($alt_path);
			$found = TRUE;
		}
	}
	else
	{
		foreach (get_instance()->load->get_package_paths(TRUE) as $package_path)
		{
			$package_path .= 'language/'.$idiom.'/'.$langfile;
			if ($basepath !== $package_path && file_exists($package_path))
			{
				include($package_path);
				$found = TRUE;
				break;
			}
		}
	}

	
	if ($found !== TRUE)
	{
		show_error('Unable to load the requested language file: language/'.$idiom.'/'.$langfile);
	}

	// 语言配置文件异常
	if ( ! isset($lang) OR ! is_array($lang))
	{
		log_message('error', 'Language file contains no data: language/'.$idiom.'/'.$langfile);

		if ($return === TRUE)
		{
			return array();
		}
		return;
	}
	if ($return === TRUE)
	{
		return $lang;
	}

	// 标记已加载
	$this->is_loaded[$langfile] = $idiom;

	// 合并语言配置项
	$this->language = array_merge($this->language, $lang);

	log_message('info', 'Language file loaded: language/'.$idiom.'/'.$langfile);
	return TRUE;
}
```

这一段代码本身没什么难以理解的，但是我个人认为```$add_suffix```、```$return```这两个参数实现的功能有点多余。```$add_suffix```是用来规格化文件名的，但是文件名这种就应该直接约定好不要搞乱七八糟的事情。```$return```返回加载的语言配置项数组，这个感觉需求不大。我感觉最奇怪的是竟然没有一个特定的```$_ci_lang_paths```。


加载后的语言配置项合并到```$language```属性中，所以要找其中一项就是从这个数组中查找。

```php
public function line($line, $log_errors = TRUE)
{
	$value = isset($this->language[$line]) ? $this->language[$line] : FALSE;

	if ($value === FALSE && $log_errors === TRUE)
	{
		log_message('error', 'Could not find the language line "'.$line.'"');
	}

	return $value;
}
```
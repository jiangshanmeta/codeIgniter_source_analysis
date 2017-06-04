加载view要完成以下几件事：

* 找到对应的view文件
* 把数据传给view
* 根据第三个参数决定是将view层以字符串形式返回，还是交给输出类

```php
public function view($view, $vars = array(), $return = FALSE)
{
	return $this->_ci_load(array('_ci_view' => $view, '_ci_vars' => $this->_ci_prepare_view_vars($vars), '_ci_return' => $return));
}

// 预处理传给视图的数据
protected function _ci_prepare_view_vars($vars)
{
	// 保证数据是array的形式
	if ( ! is_array($vars))
	{
		$vars = is_object($vars)
			? get_object_vars($vars)
			: array();
	}

	// 保证私有变量名不被占用
	foreach (array_keys($vars) as $key)
	{
		if (strncmp($key, '_ci_', 4) === 0)
		{
			unset($vars[$key]);
		}
	}

	return $vars;
}
```

view方法只是个入口，实际上是调用的```_ci_load```方法，在调用```_ci_load```之前首先预处理传给视图的数据，保证数组的形式。


首先解决第一个问题，找到相应的view文件：

```php

// _ci_load参数只有$_ci_data，打散这个数组
foreach (array('_ci_view', '_ci_vars', '_ci_path', '_ci_return') as $_ci_val)
{
	$$_ci_val = isset($_ci_data[$_ci_val]) ? $_ci_data[$_ci_val] : FALSE;
}

$file_exists = FALSE;

// _ci_load方法是 view和file两个公用方法公用的，区别是是否有$_ci_path或者$_ci_view
if (is_string($_ci_path) && $_ci_path !== '')
{
	$_ci_x = explode('/', $_ci_path);
	$_ci_file = end($_ci_x);
}
else
{
	// 确定文件名，默认扩展名是php
	$_ci_ext = pathinfo($_ci_view, PATHINFO_EXTENSION);
	$_ci_file = ($_ci_ext === '') ? $_ci_view.'.php' : $_ci_view;

	// 遍历_ci_view_paths寻找view文件
	foreach ($this->_ci_view_paths as $_ci_view_file => $cascade)
	{
		if (file_exists($_ci_view_file.$_ci_file))
		{
			$_ci_path = $_ci_view_file.$_ci_file;
			$file_exists = TRUE;
			break;
		}

		// 至今没想明白
		if ( ! $cascade)
		{
			break;
		}
	}
}

if ( ! $file_exists && ! file_exists($_ci_path))
{
	show_error('Unable to load the requested file: '.$_ci_file);
}
```

以上就是寻找view文件对应的源码，然而那个```$cascade```至今不理解起到了什么作用，文档里提过但是不知所云。


然后我们要解决把数据传递给view层，之前提到的数据预处理算是一部分，剩下的相关代码如下：

```php
// 把controller实例上的属性挂载到加载器实例上
$_ci_CI =& get_instance();
foreach (get_object_vars($_ci_CI) as $_ci_key => $_ci_var)
{
	if ( ! isset($this->$_ci_key))
	{
		$this->$_ci_key =& $_ci_CI->$_ci_key;
	}
}

// 将数据与旧数据合并，挂载到_ci_cached_vars上
empty($_ci_vars) OR $this->_ci_cached_vars = array_merge($this->_ci_cached_vars, $_ci_vars);

// 将数据数组打散
extract($this->_ci_cached_vars);
```

关于这个```_ci_cached_vars```要多说几句，这个属性用来存放我们传递给view层的数据，在加载器中还有几个相关接口用于操作这个属性：

```php
// 添加缓存的数据
public function vars($vars, $val = '')
{
	$vars = is_string($vars)
		? array($vars => $val)
		: $this->_ci_prepare_view_vars($vars);

	foreach ($vars as $key => $val)
	{
		$this->_ci_cached_vars[$key] = $val;
	}

	return $this;
}
// 清除缓存的数据
public function clear_vars()
{
	$this->_ci_cached_vars = array();
	return $this;
}
// 获得特定的缓存数据
public function get_var($key)
{
	return isset($this->_ci_cached_vars[$key]) ? $this->_ci_cached_vars[$key] : NULL;
}
// 获得全部缓存数据
public function get_vars()
{
	return $this->_ci_cached_vars;
}
```

最后就是加载并处理view文件：

```php

// 开始输出缓存
ob_start();

// 加载view文件
if ( ! is_php('5.4') && ! ini_get('short_open_tag') && config_item('rewrite_short_tags') === TRUE)
{
	echo eval('?>'.preg_replace('/;*\s*\?>/', '; ?>', str_replace('<?=', '<?php echo ', file_get_contents($_ci_path))));
}
else
{
	include($_ci_path); 
}

log_message('info', 'File loaded: '.$_ci_path);

// 以字符串形式返回view层
if ($_ci_return === TRUE)
{
	$buffer = ob_get_contents();
	@ob_end_clean();
	return $buffer;
}


if (ob_get_level() > $this->_ci_ob_level + 1)
{
	// 允许view嵌套，逐层返回结果
	ob_end_flush();
}
else
{
	// 将view层数据交给Output类处理
	$_ci_CI->output->append_output(ob_get_contents());
	@ob_end_clean();
}
```
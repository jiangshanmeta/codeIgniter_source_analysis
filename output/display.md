在了解了CI与HTTP相关方法以及与缓存相关方法之后，我们终于要看输出是如何实现的了。

## append_output

```php
public function append_output($output)
{
	$this->final_output .= $output;
	return $this;
}
```

在加载器的view方法中，我们实际上已经调用了Output的append_output方法，属性```$final_output```是最终输出值。

操作```$final_output```属性的还有两个方法：```get_output```和```set_output```:


```php
public function get_output()
{
	return $this->final_output;
}

public function set_output($output)
{
	$this->final_output = $output;
	return $this;
}
```

## _display

```php
public function _display($output = '')
{
	// 输出缓存时调用该方法，不存在controller实例，因而使用load_class获取一些相关实例
	$BM =& load_class('Benchmark', 'core');
	$CFG =& load_class('Config', 'core');

	if (class_exists('CI_Controller', FALSE))
	{
		$CI =& get_instance();
	}

	// 默认输出属性$final_output
	if ($output === '')
	{
		$output =& $this->final_output;
	}


	// 处理写缓存，我们在缓存那一节已经提过了
	if ($this->cache_expiration > 0 && isset($CI) && ! method_exists($CI, '_output'))
	{
		$this->_write_cache($output);
	}


	// 处理测试类输出的伪变量，包括所用时间和所用内存
	$elapsed = $BM->elapsed_time('total_execution_time_start', 'total_execution_time_end');
	if ($this->parse_exec_vars === TRUE)
	{
		$memory	= round(memory_get_usage() / 1024 / 1024, 2).'MB';
		$output = str_replace(array('{elapsed_time}', '{memory_usage}'), array($elapsed, $memory), $output);
	}


	if (isset($CI)
		&& $this->_compress_output === TRUE
		&& isset($_SERVER['HTTP_ACCEPT_ENCODING']) && strpos($_SERVER['HTTP_ACCEPT_ENCODING'], 'gzip') !== FALSE)
	{
		ob_start('ob_gzhandler');
	}


	// 输出响应头
	if (count($this->headers) > 0)
	{
		foreach ($this->headers as $header)
		{
			@header($header[0], $header[1]);
		}
	}


	// 使用缓存时会在这一步输出最终结果
	if ( ! isset($CI))
	{

		if ($this->_compress_output === TRUE)
		{
			if (isset($_SERVER['HTTP_ACCEPT_ENCODING']) && strpos($_SERVER['HTTP_ACCEPT_ENCODING'], 'gzip') !== FALSE)
			{
				header('Content-Encoding: gzip');
				header('Content-Length: '.strlen($output));
			}
			else
			{
				// 处理缓存文件是gzip压缩文件但是客户端不支持gzip压缩
				$output = gzinflate(substr($output, 10, -8));
			}
		}

		echo $output;
		log_message('info', 'Final output sent to browser');
		log_message('debug', 'Total execution time: '.$elapsed);
		return;
	}

	
	// 程序分析器
	if ($this->enable_profiler === TRUE)
	{
		$CI->load->library('profiler');
		if ( ! empty($this->_profiler_sections))
		{
			$CI->profiler->set_sections($this->_profiler_sections);
		}
		$output = preg_replace('|</body>.*?</html>|is', '', $output, -1, $count).$CI->profiler->run();
		if ($count > 0)
		{
			$output .= '</body></html>';
		}
	}

	// 如果controller实例有_output方法，用该方法处理要输出的结果
	if (method_exists($CI, '_output'))
	{
		$CI->_output($output);
	}
	else
	{
		// 输出结果
		echo $output;
	}

	log_message('info', 'Final output sent to browser');
	log_message('debug', 'Total execution time: '.$elapsed);
}
```

这个方法是输出类的核心方法，两个地方会用到，一个是输出缓存文件的时候，一个是controller的方法执行完成之后。前者调用该方法时controller还没有实例化，因而不能调用```get_instance```方法获取controller实例。
上一节介绍了基本的HTTP协议知识，这里介绍在CI中和HTTP协议相关的方法。

## set_status_header

```php
if ( ! function_exists('set_status_header'))
{
	// 设定响应行
	function set_status_header($code = 200, $text = '')
	{
		if (is_cli())
		{
			return;
		}

		if (empty($code) OR ! is_numeric($code))
		{
			show_error('Status codes must be numeric', 500);
		}
		
		// 提供默认的原因短语
		if (empty($text))
		{
			is_int($code) OR $code = (int) $code;
			$stati = array(
				100	=> 'Continue',
				101	=> 'Switching Protocols',

				200	=> 'OK',
				201	=> 'Created',
				202	=> 'Accepted',
				203	=> 'Non-Authoritative Information',
				204	=> 'No Content',
				205	=> 'Reset Content',
				206	=> 'Partial Content',

				300	=> 'Multiple Choices',
				301	=> 'Moved Permanently',
				302	=> 'Found',
				303	=> 'See Other',
				304	=> 'Not Modified',
				305	=> 'Use Proxy',
				307	=> 'Temporary Redirect',

				400	=> 'Bad Request',
				401	=> 'Unauthorized',
				402	=> 'Payment Required',
				403	=> 'Forbidden',
				404	=> 'Not Found',
				405	=> 'Method Not Allowed',
				406	=> 'Not Acceptable',
				407	=> 'Proxy Authentication Required',
				408	=> 'Request Timeout',
				409	=> 'Conflict',
				410	=> 'Gone',
				411	=> 'Length Required',
				412	=> 'Precondition Failed',
				413	=> 'Request Entity Too Large',
				414	=> 'Request-URI Too Long',
				415	=> 'Unsupported Media Type',
				416	=> 'Requested Range Not Satisfiable',
				417	=> 'Expectation Failed',
				422	=> 'Unprocessable Entity',
				426	=> 'Upgrade Required',
				428	=> 'Precondition Required',
				429	=> 'Too Many Requests',
				431	=> 'Request Header Fields Too Large',

				500	=> 'Internal Server Error',
				501	=> 'Not Implemented',
				502	=> 'Bad Gateway',
				503	=> 'Service Unavailable',
				504	=> 'Gateway Timeout',
				505	=> 'HTTP Version Not Supported',
				511	=> 'Network Authentication Required',
			);

			if (isset($stati[$code]))
			{
				$text = $stati[$code];
			}
			else
			{
				show_error('No status text available. Please check your status code number or supply your own message text.', 500);
			}
		}

		if (strpos(PHP_SAPI, 'cgi') === 0)
		{
			header('Status: '.$code.' '.$text, TRUE);
			return;
		}

		// 响应行的协议版本
		$server_protocol = (isset($_SERVER['SERVER_PROTOCOL']) && in_array($_SERVER['SERVER_PROTOCOL'], array('HTTP/1.0', 'HTTP/1.1', 'HTTP/2'), TRUE))
			? $_SERVER['SERVER_PROTOCOL'] : 'HTTP/1.1';

		// 设定响应行
		header($server_protocol.' '.$code.' '.$text, TRUE, $code);
	}
}
```

这个方法作用是设定响应行的三个基本信息：HTTP协议版本、状态码、原因短语。

在Output类上也有个同名方法，实质就是别名。

## get_mimes

```php
if ( ! function_exists('get_mimes'))
{
	// 获取mime对应的配置文件
	function &get_mimes()
	{
		static $_mimes;

		if (empty($_mimes))
		{
			$_mimes = file_exists(APPPATH.'config/mimes.php')
				? include(APPPATH.'config/mimes.php')
				: array();

			if (file_exists(APPPATH.'config/'.ENVIRONMENT.'/mimes.php'))
			{
				$_mimes = array_merge($_mimes, include(APPPATH.'config/'.ENVIRONMENT.'/mimes.php'));
			}
		}

		return $_mimes;
	}
}
```

这个方法和```get_config```方法基本雷同，相当于mime版的```get_config```。它把加载的结果缓存到了静态变量```$_mimes```中。这个方法会在Output类的构造函数中用到。


## set_header

```php set_header
public function set_header($header, $replace = TRUE)
{
	if ($this->_zlib_oc && strncasecmp($header, 'content-length', 14) === 0)
	{
		return $this;
	}

	// 缓存要输出的响应头
	$this->headers[] = array($header, $replace);
	return $this;
}
```

这个方法是Output类中的方法，用于设定要输出的响应头

## get_header

```php
public function get_header($header)
{
	// Combine headers already sent with our batched headers
	$headers = array_merge(
		array_map('array_shift', $this->headers),
		headers_list()
	);

	if (empty($headers) OR empty($header))
	{
		return NULL;
	}

	for ($c = count($headers) - 1; $c > -1; $c--)
	{
		if (strncasecmp($header, $headers[$c], $l = self::strlen($header)) === 0)
		{
			return trim(self::substr($headers[$c], $l+1));
		}
	}

	return NULL;
}
```

这个方法用来获取某个特定的响应头。```headers_list```方法用来获取已经输出的响应头字段，它与我们缓存在属性```$headers```中的字段合并后得到当前的响应头部字段，然后我们遍历找到最新的响应头部字段。


## set_content_type

```php
public function set_content_type($mime_type, $charset = NULL)
{
	// mime类型 结构是 主类型/子类型 ，如果传入的$mime_type不含 / 则需要根据配置文件设定
	if (strpos($mime_type, '/') === FALSE)
	{
		$extension = ltrim($mime_type, '.');
		if (isset($this->mimes[$extension]))
		{
			// get_mimes方法的结果，缓存到了Output类的$mimes中
			$mime_type =& $this->mimes[$extension];

			if (is_array($mime_type))
			{
				$mime_type = current($mime_type);
			}
		}
	}

	$this->mime_type = $mime_type;

	if (empty($charset))
	{
		$charset = config_item('charset');
	}

	$header = 'Content-Type: '.$mime_type
		.(empty($charset) ? '' : '; charset='.$charset);

	// 最终是缓存到Output的$headers上
	$this->headers[] = array($header, TRUE);
	return $this;
}
```

## get_content_type

```php
public function get_content_type()
{
	for ($i = 0, $c = count($this->headers); $i < $c; $i++)
	{
		if (sscanf($this->headers[$i][0], 'Content-Type: %[^;]', $content_type) === 1)
		{
			return $content_type;
		}
	}

	return 'text/html';
}
```

这个方法返回当前使用的mime类型，与```set_content_type```方法相呼应，但问题是我们已经使用属性```$mime_type```来表明当前类型了，为啥不直接返回该属性呢？



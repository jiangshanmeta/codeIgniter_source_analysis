在CI中，默认是不使用缓存的，要使用缓存，我们需要手动调用output的```cache```方法

```php
public function cache($time)
{
	$this->cache_expiration = is_numeric($time) ? $time : 0;
	return $this;
}
```

这个方法以分钟为单位设定了缓存的时间。

当使用output的```_display```方法输出页面时，会检查这个缓存时间，生成缓存文件。

```php
if ($this->cache_expiration > 0 && isset($CI) && ! method_exists($CI, '_output'))
{
	$this->_write_cache($output);
}
```

我们来具体看下如何缓存输出内容的：

```php
public function _write_cache($output)
{
	$CI =& get_instance();

	// 确定缓存输出路径
	$path = $CI->config->item('cache_path');
	$cache_path = ($path === '') ? APPPATH.'cache/' : $path;
	if ( ! is_dir($cache_path) OR ! is_really_writable($cache_path))
	{
		log_message('error', 'Unable to write cache file: '.$cache_path);
		return;
	}

	// 确定缓存文件名
	$uri = $CI->config->item('base_url')
		.$CI->config->item('index_page')
		.$CI->uri->uri_string();
	if (($cache_query_string = $CI->config->item('cache_query_string')) && ! empty($_SERVER['QUERY_STRING']))
	{
		if (is_array($cache_query_string))
		{
			$uri .= '?'.http_build_query(array_intersect_key($_GET, array_flip($cache_query_string)));
		}
		else
		{
			$uri .= '?'.$_SERVER['QUERY_STRING'];
		}
	}
	// 缓存文件名md5加密
	$cache_path .= md5($uri);

	// 获取文件句柄
	if ( ! $fp = @fopen($cache_path, 'w+b'))
	{
		log_message('error', 'Unable to write cache file: '.$cache_path);
		return;
	}

	if (flock($fp, LOCK_EX))
	{
		// 处理需要压缩的情况，采用gzip压缩
		if ($this->_compress_output === TRUE)
		{
			$output = gzencode($output);

			if ($this->get_header('content-type') === NULL)
			{
				$this->set_content_type($this->mime_type);
			}
		}

		$expire = time() + ($this->cache_expiration * 60);

		// 在缓存文件中写入过期时间和响应头信息
		$cache_info = serialize(array(
			'expire'	=> $expire,
			'headers'	=> $this->headers
		));

		$output = $cache_info.'ENDCI--->'.$output;

		// 最终将内容写入到缓存文件
		for ($written = 0, $length = self::strlen($output); $written < $length; $written += $result)
		{
			if (($result = fwrite($fp, self::substr($output, $written))) === FALSE)
			{
				break;
			}
		}

		flock($fp, LOCK_UN);
	}
	else
	{
		log_message('error', 'Unable to secure a file lock for file at: '.$cache_path);
		return;
	}

	fclose($fp);

	
	if (is_int($result))
	{
		// 写入成功，设定文件权限
		chmod($cache_path, 0640);
		log_message('debug', 'Cache file written: '.$cache_path);

		$this->set_cache_header($_SERVER['REQUEST_TIME'], $expire);
	}
	else
	{
		// 写入失败，删除文件
		@unlink($cache_path);
		log_message('error', 'Unable to write the complete cache content at: '.$cache_path);
	}
}
```

写完本地缓存文件之后，还要设定对应的HTTP响应字段：

```php
public function set_cache_header($last_modified, $expiration)
{
	$max_age = $expiration - $_SERVER['REQUEST_TIME'];

	// 如果满足 条件请求，304即可
	if (isset($_SERVER['HTTP_IF_MODIFIED_SINCE']) && $last_modified <= strtotime($_SERVER['HTTP_IF_MODIFIED_SINCE']))
	{
		$this->set_status_header(304);
		exit;
	}
	else
	{
		// 设定缓存必要的HTTP响应头
		header('Pragma: public');
		header('Cache-Control: max-age='.$max_age.', public');
		header('Expires: '.gmdate('D, d M Y H:i:s', $expiration).' GMT');
		header('Last-modified: '.gmdate('D, d M Y H:i:s', $last_modified).' GMT');
	}
}
```

这样在缓存时间内，收到这份报文的浏览器默认会使用本地缓存文件，不会发起HTTP请求，强行刷新时会发起条件请求。当其他人按照相同的uri访问时，会先尝试输出缓存：

```php
// 在CodeIgniter文件中，实例化Output类之后
if ($EXT->call_hook('cache_override') === FALSE && $OUT->_display_cache($CFG, $URI) === TRUE)
{
	exit;
}
```

我们看一下是如何使用本地缓存的：

```php
public function _display_cache(&$CFG, &$URI)
{
	// 确定缓存文件路径
	$cache_path = ($CFG->item('cache_path') === '') ? APPPATH.'cache/' : $CFG->item('cache_path');
	$uri = $CFG->item('base_url').$CFG->item('index_page').$URI->uri_string;
	if (($cache_query_string = $CFG->item('cache_query_string')) && ! empty($_SERVER['QUERY_STRING']))
	{
		if (is_array($cache_query_string))
		{
			$uri .= '?'.http_build_query(array_intersect_key($_GET, array_flip($cache_query_string)));
		}
		else
		{
			$uri .= '?'.$_SERVER['QUERY_STRING'];
		}
	}

	$filepath = $cache_path.md5($uri);

	// 查看是否有可用缓存
	if ( ! file_exists($filepath) OR ! $fp = @fopen($filepath, 'rb'))
	{
		return FALSE;
	}

	// 获取缓存内容
	flock($fp, LOCK_SH);
	$cache = (filesize($filepath) > 0) ? fread($fp, filesize($filepath)) : '';
	flock($fp, LOCK_UN);
	fclose($fp);

	// 从缓存文件中获取过期时间、响应头等信息
	if ( ! preg_match('/^(.*)ENDCI--->/', $cache, $match))
	{
		return FALSE;
	}
	$cache_info = unserialize($match[1]);
	$expire = $cache_info['expire'];

	$last_modified = filemtime($filepath);

	// 如果缓存已过期，删掉缓存文件，走非缓存的路
	if ($_SERVER['REQUEST_TIME'] >= $expire && is_really_writable($cache_path))
	{
		@unlink($filepath);
		log_message('debug', 'Cache file has expired. File deleted.');
		return FALSE;
	}
	else
	{
		// 设定缓存相关的响应头
		$this->set_cache_header($last_modified, $expire);
	}

	// 设定缓存文件保留的响应头
	foreach ($cache_info['headers'] as $header)
	{
		$this->set_header($header[0], $header[1]);
	}

	// 输出缓存内容
	$this->_display(self::substr($cache, self::strlen($match[0])));
	log_message('debug', 'Cache file is current. Sending it to browser.');
	return TRUE;
}
```

首先是确定缓存文件的路径，这一块其实可以单独封成一个小函数，毕竟写```_write_cache```和```_display_cache```都用到了，找到缓存文件之后，解析出文件的过期时间和缓存的响应头，如果过期了就删除文件，然后设定缓存相关的响应头信息，最后输出缓存。


与缓存相关的最后一个方法是删除缓存：

```php
public function delete_cache($uri = '')
{
	$CI =& get_instance();
	$cache_path = $CI->config->item('cache_path');
	if ($cache_path === '')
	{
		$cache_path = APPPATH.'cache/';
	}

	if ( ! is_dir($cache_path))
	{
		log_message('error', 'Unable to find cache path: '.$cache_path);
		return FALSE;
	}

	if (empty($uri))
	{
		$uri = $CI->uri->uri_string();

		if (($cache_query_string = $CI->config->item('cache_query_string')) && ! empty($_SERVER['QUERY_STRING']))
		{
			if (is_array($cache_query_string))
			{
				$uri .= '?'.http_build_query(array_intersect_key($_GET, array_flip($cache_query_string)));
			}
			else
			{
				$uri .= '?'.$_SERVER['QUERY_STRING'];
			}
		}
	}

	$cache_path .= md5($CI->config->item('base_url').$CI->config->item('index_page').ltrim($uri, '/'));

	if ( ! @unlink($cache_path))
	{
		log_message('error', 'Unable to delete cache file for '.$uri);
		return FALSE;
	}

	return TRUE;
}
```

删除缓存的逻辑很简单，只需要找到相应文件删掉就好了。这里最主要的是处理文件路径及文件名，所以，为啥相关代码不拆分出来呢？
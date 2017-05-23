日期辅助函数包含了一些处理日期的函数。我个人认为PHP已经提供了足够的日期处理函数了，至少比javascript强。

## now([$timezone = NULL])

基于时区返回当前时间戳。

```php
if ( ! function_exists('now'))
{
	function now($timezone = NULL)
	{
		if (empty($timezone))
		{
			$timezone = config_item('time_reference');
		}

		if ($timezone === 'local' OR $timezone === date_default_timezone_get())
		{
			return time();
		}

		$datetime = new DateTime('now', new DateTimeZone($timezone));
		sscanf($datetime->format('j-n-Y G:i:s'), '%d-%d-%d %d:%d:%d', $day, $month, $year, $hour, $minute, $second);

		return mktime($hour, $minute, $second, $month, $day, $year);
	}
}
```

如果不修改配置文件，默认条件下就是直接调用```time```函数。考虑时区的话就需要调用```DateTimeZone```这个类了。



## nice_date([$bad_date = ''[, $format = FALSE]])

这个函数我觉得功能非常强大，它将非格式化的日期字符串格式化处理，默认处理格式为时间戳。


```php
if ( ! function_exists('nice_date'))
{
	function nice_date($bad_date = '', $format = FALSE)
	{
		if (empty($bad_date))
		{
			return 'Unknown';
		}
		// 默认处理格式
		elseif (empty($format))
		{
			$format = 'U';
		}

		// 处理YYYYMM和MMYYYY的情况
		if (preg_match('/^\d{6}$/i', $bad_date))
		{
			if (in_array(substr($bad_date, 0, 2), array('19', '20')))
			{
				$year  = substr($bad_date, 0, 4);
				$month = substr($bad_date, 4, 2);
			}
			else
			{
				$month  = substr($bad_date, 0, 2);
				$year   = substr($bad_date, 2, 4);
			}

			return date($format, strtotime($year.'-'.$month.'-01'));
		}

		// 处理YYYYMMDD的情况
		if (preg_match('/^\d{8}$/i', $bad_date, $matches))
		{
			return DateTime::createFromFormat('Ymd', $bad_date)->format($format);
		}

		// 处理MM-DD-YYYY或者M-D-YYYY的情况
		if (preg_match('/^(\d{1,2})-(\d{1,2})-(\d{4})$/i', $bad_date, $matches))
		{
			return date($format, strtotime($matches[3].'-'.$matches[1].'-'.$matches[2]));
		}

		
		if (date('U', strtotime($bad_date)) === '0')
		{
			return 'Invalid Date';
		}

		return date($format, strtotime($bad_date));
	}
}
```

PHP的```strtotime```功能非常强大了，这里相当于是对其包装了一层，修复了几个特殊情况。然而CI已经把这个API给deprecated。其实这个小函数还有个特殊情况没考虑，如果我传入的就是个时间戳呢？这时候strtotime是无法正常处理的。既然作者已经deprecated这个API了，提个issue也没什么意思(虽然我对这个API的bug提过issue)，自己攒的代码库里处理时间的时候要注意这点。

## days_in_month([$month = 0[, $year = '']])

返回月份中的天数。看到这个描述我想到的是使用date函数，第一个参数传```"t"```即可，然而作者使用了```cal_days_in_month```这个方法。看了一眼手册我怀疑这个方法就是对前者的封装，因为有人提出了这么个polyfill：

```php
if (!function_exists('cal_days_in_month')) 
{ 
    function cal_days_in_month($calendar, $month, $year) 
    { 
        return date('t', mktime(0, 0, 0, $month, 1, $year)); 
    } 
} 
if (!defined('CAL_GREGORIAN')) 
    define('CAL_GREGORIAN', 1); 
```

结合```days_in_month```的实现我怀疑这个方法考虑到了1970年以前，然而日常开发基本不需要考虑这个。

```php
if ( ! function_exists('days_in_month'))
{
	function days_in_month($month = 0, $year = '')
	{
		if ($month < 1 OR $month > 12)
		{
			return 0;
		}
		elseif ( ! is_numeric($year) OR strlen($year) !== 4)
		{
			$year = date('Y');
		}

		// 尽可能使用系统提供的cal_days_in_month方法
		if (defined('CAL_GREGORIAN'))
		{
			return cal_days_in_month(CAL_GREGORIAN, $month, $year);
		}

		// date的方案似乎只能处理1970年以后
		if ($year >= 1970)
		{
			return (int) date('t', mktime(12, 0, 0, $month, 1, $year));
		}

		// 1970年以前闰年的2月
		if ($month == 2)
		{
			if ($year % 400 === 0 OR ($year % 4 === 0 && $year % 100 !== 0))
			{
				return 29;
			}
		}

		$days_in_month	= array(31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31);
		return $days_in_month[$month - 1];
	}
}
```




关于日期处理函数，我也结合日常开发[封装了一堆方法](https://github.com/jiangshanmeta/utility/tree/master/php/libraries/moment)，我是直接将方法组织在一个类中(类名叫Moment，就是因为js有个日期处理库叫moment)。
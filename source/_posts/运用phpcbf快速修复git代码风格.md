---
title: 运用phpcbf快速修复git代码风格
date: 2017-06-18 16:21:16
tags: 
  - git
  - phpcbf
  - filter-branch
  - tree-filter
category: tech
---

由于历史原因，团队之前对于php代码标准没有深入的认识，因此以前对于代码风格这件事就放羊了。后来，随着团队规模的扩大，我们逐步意识到了代码风格不同意对于整体合作的不利影响，再加上我们对于代码加入了review机制，看到各种各样不同风格的代码，总是会对代码本身意图的理解造成一定程度的障碍。在这样的背景下，我们开始在代码加入了强制性的标准——PSR2。而对于命名空间，由于老代码根本就没有考虑这块，而且命名空间的影响也不大，因此我们放弃了这部分标准。



我们对于代码风格的实施，采用了一种灰度的方式实现。即对修改的代码，削减了部分代码标准，而对新加的代码，标准则相对严格一些。

最近由于业务需要，对一些线下项目有了较多的接触，发现其代码风格已经是颇难接受了。一些代码大小写不统一，格式不统一，最普遍的问题，则是`\t`的使用。因此萌发将这个项目的整体风格都修正一下的想法。
<!--more-->

## 基本思路

因为我们代码风格的控制是基于phpcs的，因此修复代码的任务就交给phpcbf了。最简单的办法，最简单的命令就是

```shell
phpcbf ./ --standard=PSR2 --extensions=php
```

这样就能对整体代码进行整体修复了。

不过，问题也是显而易见的，经过这样一次修复，在这次commit之后的代码再要通过`git blame`查询某行代码的修改者的话，就不再那么简单了。这也是之前的项目做完这样的修复之后遇到的最显著问题之一。因此，最好能避免掉这个问题。

那么，比较彻底的方式，当然就是对所有历史版本的文件都用phpcbf处理一次，这样就能让git库的所有历史版本都保持基本的代码规范（phpcbf只能处理简单的修复工作，并不能修复所有的有问题的代码），而且原来所有的历史变动信息也还在。当然，也有副作用，各个commit的版本号都会出现变动。不过，这个副作用比起前面git blame的问题，可以忽略了。

## v1.0.0
最简单最直接也是最容易想到的办法，就是用`git filter-branch`命令，然后再用`--tree-filter`选项对整个历史代码格式化一下。代码如下：

```shell
git filter-branch --tree-filter "phpcbf ./ --standard=PSR2 --extensions=php" --prune-empty -f
```

这个命令里边，`filter-branch`这个命令对于git而言，算是一个核武器了，因为它可以通过脚本的方式对大量提交进行修改。比如，全局地修改邮箱地址，或者把某一个文件从每一个提交中都删除掉。

`--tree-filter`选项会把所有历史提交版本依次提取到指定目录里（默认是`.git-rewrite`，也可以通过-d选项指定此目录），然后通过此选项的值对应的命令对这些提交进行处理。

`--prune-empty`选项的话，会将重写历史后一些没有任何修改内容的commit直接去掉。也就是说，原来有1000个commit以后，加上这个选项后，有可能变成<1000个提交。

`-f` 如果临时目录已经存在，git会拒绝执`git filter-branch`命令，加上此命令后，则会强制执行此命令。

好，开始执行了，这个项目约有1200个commit，所以整体跑下来还真的是需要蛮长时间的。刚开始，由于代码量有限，还蛮快的，几秒钟就处理完了一个commit，但是越到后面，每次commit处理的时间就越长。不过，这个还不是主要的问题，主要的问题在于这样执行过程中，遇到一些有报错的，phpcbf进程的退出码不为0，会导致git中断处理流程。所以，得包装一下这个命令，使其退出码一直为0。因此，出现了升级版v1.0.1。

## v1.0.1
写一个脚本phpcbf.sh。
```shell
#! /bin/bash
phpcbf ./ --standard=PSR2 --extensions=php
exit 0
```
同时，调用命令变成了：
```shell
git filter-branch --tree-filter "sh phpcbf.sh" --prune-empty -f
```

这样一来，能保障及时phpcbf执行过程中即使有报错，也不会影响程序继续执行。不过，没有等执行完毕，我就中止了这个命令。因为，要跑完1200多个commit，还真得花上好几个小时，太耗时间了。怎么办呢？在不改动代码前提的基础下，我的想法是把代码放到内存里去执行。

## v1.0.2
怎么放内存呢？听起来好玄乎，但是在linux系统上，却是很简单的事情——把代码放到`/dev/shm/`这个目录就行了。用`df`命令就可以看到这个目录所在了：

```shell
$: df -h
Filesystem                     Size  Used Avail Use% Mounted on
/dev/mapper/vg_livecd-lv_root   50G   28G   21G  58% /
tmpfs                          1.9G     0  1.9G   0% /dev/shm
/dev/sda1                      485M   40M  420M   9% /boot
/dev/mapper/vg_livecd-lv_home  439G  264G  153G  64% /home
/dev/mapper/lvm--data-data     483G   83G  375G  19% /data
```

/dev/shm/是linux下一个目录，/dev/shm目录不在磁盘上，而是在内存里，因此使用linux /dev/shm/的效率非常高，直接写进内存。对于磁盘io不高的虚拟机，把文件IO较多的东东丢到这个目录来处理，还是蛮适合的。事实证明，这个也确实比在硬盘的目录里边处理要快很多。

不过，鉴于处理这个问题的时候，已经很晚了，而且，这样处理起来效率还是很低，所以，我就直接让其在机器里边跑着，睡觉去了。
```shell
nohup git filter-branch --tree-filter "sh phpcbf.sh" --prune-empty -f &
```
`nohup`命令可以保证，即使ssh断掉，这个进程也会继续执行，`&`则是表示进程进入后台运行。

就算是用了内存，就算是跑了好几个小时，早上我起来的时候一看，程序也还在跑着呢，看起来，这样的机制还是得来个大的升级才行。

## v2.0.0
进入到2.0版本，是因为，我把原来的单进程的方式，修改成了多进程的模式。毕竟，phpcbf进程对于每个文件而言，都是独立运行的，不会对彼此有相互依赖，因此，升级到多进程模式是没有任何问题的，只需要考虑好使用过少个进程，怎么给它们派活就行了。废话少说，直接上代码。

```php
#! /home/work/php/bin/php
<?php
$curdir = getcwd();
// 找出层级为2的目录，分别配置一个进程予以处理，在层级1目录的文件，后续单独操作
exec(sprintf('find %s -maxdepth 2 -mindepth 2 -type d', $curdir), $dirs);
foreach($dirs as $key => $dir) {
    $dir = trim($dir);
    if (!$dir || $dir == '.' || $dir == '..') {
        unset($dirs[$key]);
    }

    // phpcbf处理的时候，如果没有任何一个可用文件，会报错，因此提前检查
    $cmd = sprintf("find %s -type f -name *.php|wc -l", $dir);
    ob_start();
    $count = system($cmd);
    ob_get_clean();
    if (!$count) {
        unset($dirs[$key]);
    }
}
$dirs = array_values($dirs);
for ($i = 1, $l = count($dirs); $i < $l; ++$i) { 
    $pid = pcntl_fork(); 

    if (!$pid) { 
        $dealDir = $dirs[$i];
        print "In child $i\n"; 
        $cmd = sprintf("phpcbf --standard=PSR2 --extensions=php %s", $dealDir);
        echo $cmd.PHP_EOL;

        system($cmd);

        exit($i); 
    } 
} 

while (pcntl_waitpid(0, $status) != -1) { 
    $status = pcntl_wexitstatus($status); 
    echo "Child $status completed\n"; 
} 
```

这样一来，多进程模式开始处理了，效率一下子上来了，原来五六个小时处理的事情，现在也能缩短了一个小时以内了。

### 处理剩余的单个文件
前面处理的是二级目录里边的文件，还剩下一些单独的一级目录的文件，因为只有一个文件，所以直接在每个commit里边对单个文件应用phpcbf处理就行了。不过，这样下来，效率也不高，往往也需要几分钟才能处理完1个文件。后来，又总结了一下，提取出一种新的处理模式。

考虑到同一个文件在整个git历史了，实际上改动的次数不多，而且，如果原始内容一样，那么每次应用phpcbf之后，结果应该是一样的。因此，考虑通过缓存phpcbf结果的方式来加快处理速度。直接看代码吧：

```php
<?php
// 为了更好地知道过程中出现的问题，把错误级别提高了
error_reporting(E_ALL);
ini_set('display_errors', 1);

$path = getcwd().$argv[1];
// 在早期的提交里，文件可能不存在
if (!is_file($path)) {
    exit(0);
}
// 根据内容来定义缓存文件路径
$md5 = md5_file($path);
$tmpFile = __DIR__.'/.tmp/'.$md5;
if (is_file($tmpFile)) {
    // 如果缓存过结果，直接复制过来即可，不需要再次phpcbf了
    copy($tmpFile, $path);
    exit(0);
}
// 有修改的内容的话，执行phpcbf，并将结果缓存起来
$cmd = 'phpcbf --standard=PSR2 '.$path;
system($cmd);
copy($path, $tmpFile);
exit(0);
```
通过这样一个优化，对单个文件的修改，已经可以控制到60s左右了。


### 问题多多
不过，最后看`git log --stat`的时候，还是发现有一些问题，出现了大量的*.rej文件，而且每次的提交都出现了大量文件提交。应该还是phpcbf执行出错的时候留下的文件没有及时清理导致的。而且，及时正确，这个多进程模式也还是需要小时级别才能处理好。需要再次改进。

## v3.0.0
最后，还是决定大改一次。怎么改呢？就用第2个版本里边针对单个文件处理的模式来改。因为整个git项目虽然有1200个commit之多，但是，每次对于git库的修改，实际上只涉及到几个文件，那么如果我们能将之前的代码phpcbf后的结果缓存的话，可以节省大量的时间。

```php
<?php
error_reporting(E_ALL);
ini_set('display_errors', 1);

// 找出所有的php文件
$dir = getcwd();
exec("find $dir -type f -name *.php", $phps);

foreach($phps  as $file) {
    phpfixFile($file);
}

// 获取文件的缓存路径
function getTmpFile($file)
{
    $md5 = md5_file($file);
    return __DIR__.'/.tmp/'.$md5;
}

// phpcbf一个文件
function phpfixFile($file)
{
    $tmpFile = getTmpFile($file);
    if (is_file($tmpFile)) {
        copy($tmpFile, $file);
    } else {
        system("phpcbf --standard=PSR2 $file");
        copy($file, $tmpFile);
    }
}

exit(0);
```
这样一改，所有的东西都正常了，最后下来，大概花了860s，也就是14分钟多的样子就完成了所有代码的编码规范工作。

## 总结
从v1到v3三个方案，总体时间从6个小时降低到2个小时（处理单个有问题文件费时较多），再到15分钟，终于完成了这项工作的较为完美的蜕变。当然，在v3的基础上，如果再返回去结合多进程的话，处理速度应该还可以有一个幅度的上升。只要基于实践，不断磨合，还是可以对一件事情不断地探究和改进，使其可行性和稳定性更高的境界。

最后，总结一下，对git历史进行编码规范的时候，需要注意如下几个问题：
1. 在内存里运行；
2. 重复代码缓存执行结果；
3. 多实践多推敲；





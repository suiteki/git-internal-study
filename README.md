---
title: Git 内部原理
notebook: 学习探索
tags: git
---

```
Git = 基于内容寻址的文件系统 + 版本控制系统
```

### 四种类型的对象

blob、tree、commit、tag

#### blob 类型对象

blob 对应于文件。新建 Git 目录并初始化，可以看到当前并无对象

```
~/Documents/Github/git-object » git init
Initialized empty Git repository in /Users/suiteki/Documents/Github/git-object/.git/
------------------------------------------------------------
~/Documents/Github/git-object(branch:master) » find .git/objects -type f
```

使用底层命令`git hash-object`直接创建 blob 并用`git cat-file`查看类型和内容。

```
~/Documents/Github/git-object(branch:master) » echo "object 1" | git hash-object -w --stdin
c9c0bfeb224cb32ab143d30a364bec1dd2cc9d95
------------------------------------------------------------
~/Documents/Github/git-object(branch:master) » find .git/objects -type f
.git/objects/c9/c0bfeb224cb32ab143d30a364bec1dd2cc9d95
------------------------------------------------------------
~/Documents/Github/git-object(branch:master) » git cat-file -t c9c0bfe
blob
------------------------------------------------------------
~/Documents/Github/git-object(branch:master) » git cat-file -p c9c0bfe
object 1
```

新建文件并修改，同样用`git hash-object`创建blob，发现多出了两个对象，可见 Git 存储的是整个文件而不是增量（就松散对象而言）

```
~/Documents/Github/git-object(branch:master) » echo "object 2" > file
------------------------------------------------------------
~/Documents/Github/git-object(branch:master*) » git hash-object -w file
77e38eb8ed95e8e49a5b8597629d8f00312f4f5a
------------------------------------------------------------
~/Documents/Github/git-object(branch:master*) » find .git/objects -type f
.git/objects/77/e38eb8ed95e8e49a5b8597629d8f00312f4f5a
.git/objects/c9/c0bfeb224cb32ab143d30a364bec1dd2cc9d95
------------------------------------------------------------
~/Documents/Github/git-object(branch:master*) » echo "object 3" > file
------------------------------------------------------------
~/Documents/Github/git-object(branch:master*) » git hash-object -w file
0dd845d1604b0a93c49d022ac003e0c71e825ff2
------------------------------------------------------------
~/Documents/Github/git-object(branch:master*) » find .git/objects -type f
.git/objects/0d/d845d1604b0a93c49d022ac003e0c71e825ff2
.git/objects/77/e38eb8ed95e8e49a5b8597629d8f00312f4f5a
.git/objects/c9/c0bfeb224cb32ab143d30a364bec1dd2cc9d95
```

#### tree 类型对象

tree 对应于目录，内部可包含有 tree（子目录）和 blob（文件）。这里使用`update-index`将 file 添加到暂存区域，用`write-tree`将暂存区域的内容写到一个 tree 对象（此时存在4个对象）。

```
~/Documents/Github/git-object(branch:master*) » git update-index --add file
------------------------------------------------------------
~/Documents/Github/git-object(branch:master*) » git write-tree
440c92d74bae1cce922d2392621086a84839b55f
------------------------------------------------------------
~/Documents/Github/git-object(branch:master*) » git cat-file -p 440c92d
100644 blob 0dd845d1604b0a93c49d022ac003e0c71e825ff2    file
```

继续添加目录和文件，同样的方法添加至暂存并写到 tree（此时有7个对象），查看该 tree 并递归可得到一个完整的目录结构。`read-tree`可将内容读出替换整个暂存区，或者用`--prefix`将其作为一个子 tree 读到暂存区。

```
~/Documents/Github/git-object(branch:master*) » mkdir dir1
------------------------------------------------------------
~/Documents/Github/git-object(branch:master*) » echo "file in dir1" > dir1/file-in-dir1
------------------------------------------------------------
~/Documents/Github/git-object(branch:master*) » git update-index --add dir1/file-in-dir1
------------------------------------------------------------
~/Documents/Github/git-object(branch:master*) » git write-tree
9523540db32046e57a16f4408dbdf3a2a972fd32
------------------------------------------------------------
~/Documents/Github/git-object(branch:master*) » git cat-file -p 9523540
040000 tree 99b20f755c8a258cd18a443c8dedfd73c480b24b    dir1
100644 blob 0dd845d1604b0a93c49d022ac003e0c71e825ff2    file
------------------------------------------------------------
~/Documents/Github/git-object(branch:master*) » git cat-file -p 99b20f7
100644 blob ed76f5a420c5c86f27e1adcea9afb4287d1cd49e    file-in-dir1
```


`git cat-file -p master^{tree}`或者` git ls-tree master`可查看master上最新commit所指tree对象的内容。

#### commit 类型对象

commit是项目的不同快照，内容包含有一个 tree（提交时的签出目录）、parent、author、committer等相关元信息。这里提交两次，第二次提交以前一个作为 parent。

```
~/Documents/Github/git-object(branch:master*) » echo "version 0.1" | git commit-tree 440c92d
f61a97dd3270e5f0cd140170bee184c2d2018906
------------------------------------------------------------
~/Documents/Github/git-object(branch:master*) » git cat-file -p f61a97d
tree 440c92d74bae1cce922d2392621086a84839b55f
author suiteki <hythyt9898@vip.qq.com> 1412771527 +0800
committer suiteki <hythyt9898@vip.qq.com> 1412771527 +0800

version 0.1
------------------------------------------------------------
~/Documents/Github/git-object(branch:master*) » echo "version 0.2" | git commit-tree 9523540 -p f61a97d
406b366b64239c963ce752e9b301aaca580c14be
------------------------------------------------------------
~/Documents/Github/git-object(branch:master*) » git cat-file -p 406b366
tree 9523540db32046e57a16f4408dbdf3a2a972fd32
parent f61a97dd3270e5f0cd140170bee184c2d2018906
author suiteki <hythyt9898@vip.qq.com> 1412771635 +0800
committer suiteki <hythyt9898@vip.qq.com> 1412771635 +0800

version 0.2
```

查看最后一次提交即可看到 Git 历史，

```
~/Documents/Github/git-object(branch:master*) » git log --stat 406b366
commit 406b366b64239c963ce752e9b301aaca580c14be
Author: suiteki <hythyt9898@vip.qq.com>
Date:   Wed Oct 8 20:33:55 2014 +0800

    version 0.2

 dir1/file-in-dir1 | 1 +
 1 file changed, 1 insertion(+)

commit f61a97dd3270e5f0cd140170bee184c2d2018906
Author: suiteki <hythyt9898@vip.qq.com>
Date:   Wed Oct 8 20:32:07 2014 +0800

    version 0.1

 file | 1 +
 1 file changed, 1 insertion(+)
```

#### tag 类型对象

上面的方式查看 log 需要指定提交ID，为了方便，可用一个简单的名字替代 SHA1 值来检索，这被称为引用（references 或 refs），首先查看现有的目录可知无任何引用。`git status`查看状态也是未提交，将最后一次提交写入`.git/refs/heads/master`之后发现工作目录变为 clean 状态，且`git log`可直接查看日志了。

```
~/Documents/Github/git-object(branch:master*) » find .git/refs
.git/refs
.git/refs/heads
.git/refs/tags
------------------------------------------------------------
~/Documents/Github/git-object(branch:master*) » echo "406b366b64239c963ce752e9b301aaca580c14be" > .git/refs/heads/master
------------------------------------------------------------
~/Documents/Github/git-object(branch:master) » git log --pretty=oneline
406b366b64239c963ce752e9b301aaca580c14be version 0.2
f61a97dd3270e5f0cd140170bee184c2d2018906 version 0.1
```

直接修改引用文件的方式并不推荐使用，Git 提供了一个安全的命令`update-ref`。这里新增一条分支，指向commit的第一版，上层指令`git branch XXX`基本就是执行`update-ref`。

```
~/Documents/Github/git-object(branch:master) » git update-ref refs/heads/bak f61a97dd3270e5f0cd140170bee184c2d2018906
------------------------------------------------------------
~/Documents/Github/git-object(branch:master) » git branch
  bak
* master
```

**HEAD** 标记用于指向当前所在分支的引用标识符，当切换分支时内容随之变化。当执行`git commit`时，新建 commit 对象的父对象就会设置为 HEAD 所指引用的 SHA1 值。同样的不推荐手动编辑 HEAD 文件，而是使用`symbolic-ref`修改。

```
~/Documents/Github/git-object(branch:master) » cat .git/HEAD
ref: refs/heads/master
------------------------------------------------------------
~/Documents/Github/git-object(branch:master) » git checkout bak
Switched to branch 'bak'
------------------------------------------------------------
~/Documents/Github/git-object(branch:bak) » cat .git/HEAD
ref: refs/heads/bak
```

tag 是最后一种对象类型，可以类比为`指向 对象 的指针常量`。 Tag 有两种类型： annotated 和 lightweight,前者会新建一个 tag 对象。两者对比如下，`git tag -a`表示创建的是 anotated 类型 TAG，`.git/refs/tags/`下内容为 tag 对象的 SHA1。

```
~/Documents/Github/git-object(branch:master) » git update-ref refs/tags/v0.1 f61a97dd3270e5f0cd140170bee184c2d2018906
------------------------------------------------------------
~/Documents/Github/git-object(branch:master) » git tag -a v0.11 f61a97dd3270e5f0cd140170bee184c2d2018906 -m "annotated tag"
~/Documents/Github/git-object(branch:master) » find .git/refs/tags
.git/refs/tags
.git/refs/tags/v0.1
.git/refs/tags/v0.11
------------------------------------------------------------
~/Documents/Github/git-object(branch:master) » cat .git/refs/tags/v0.1
f61a97dd3270e5f0cd140170bee184c2d2018906
------------------------------------------------------------
~/Documents/Github/git-object(branch:master) » cat .git/refs/tags/v0.11
674c6e1d4c29302a1c1f92d6de6dc835b405cf6c
------------------------------------------------------------
~/Documents/Github/git-object(branch:master) » git cat-file -p 674c6e1d4c29302a1c1f92d6de6dc835b405cf6c
object f61a97dd3270e5f0cd140170bee184c2d2018906
type commit
tag v0.11
tagger suiteki <hythyt9898@vip.qq.com> 1412817327 +0800

annotated tag
```

**remotes**

Remotes 引用和分支的区别在于他们不能被 checkout。这里将本地代码 push 到远端仓库即可看到本地仓库下新增 remotes 引用，内容为当前分支最新 commit。

```
~/Documents/Github » git init --bare git-origin
Initialized empty Git repository in /Users/suiteki/Documents/Github/git-origin/
------------------------------------------------------------
~/Documents/Github » cd git-object
------------------------------------------------------------
~/Documents/Github/git-object(branch:master) » git push origin master
Counting objects: 7, done.
Delta compression using up to 8 threads.
Compressing objects: 100% (3/3), done.
Writing objects: 100% (7/7), 513 bytes | 0 bytes/s, done.
Total 7 (delta 0), reused 0 (delta 0)
To /Users/suiteki/Documents/Github/git-origin
 * [new branch]      master -> master
------------------------------------------------------------
~/Documents/Github/git-object(branch:master) » find .git/refs/remotes
.git/refs/remotes
.git/refs/remotes/origin
.git/refs/remotes/origin/
------------------------------------------------------------
~/Documents/Github/git-object(branch:master) » cat .git/refs/remotes/origin/master
406b366b64239c963ce752e9b301aaca580c14be
```

切换到远端仓库可以看到 push 过来的 objects 和 refs。

```
~/Documents/Github/git-origin(branch:master) » find objects refs -type f
objects/0d/d845d1604b0a93c49d022ac003e0c71e825ff2
objects/40/6b366b64239c963ce752e9b301aaca580c14be
objects/44/0c92d74bae1cce922d2392621086a84839b55f
objects/95/23540db32046e57a16f4408dbdf3a2a972fd32
objects/99/b20f755c8a258cd18a443c8dedfd73c480b24b
objects/ed/76f5a420c5c86f27e1adcea9afb4287d1cd49e
objects/f6/1a97dd3270e5f0cd140170bee184c2d2018906
refs/heads/master
```

### 对象存储
对象数据的存储过程如下：irb 进入 Ruby 交互模式后执行，注意内容是zlib压缩后存储的。

```ruby
irb(main):001:0> content = "object content"
=> "object content"
irb(main):002:0> header = "blob #{content.length}\0"
=> "blob 14\u0000"
irb(main):003:0> store = header + content
=> "blob 14\u0000object content"
irb(main):004:0> require 'digest/sha1'
=> true
irb(main):005:0> sha1 = Digest::SHA1.hexdigest(store)
=> "5e95f9a59545b492638f77c3832b121e0aeeedbd"
irb(main):006:0> require 'zlib'
=> true
irb(main):007:0> content = Zlib::Deflate.deflate(store)
=> "x\x9CK\xCA\xC9OR04a\xC8O\xCAJM.QH\xCE\xCF+I\xCD+\x01\x00S\"\a\xB7"
irb(main):008:0> path = '.git/objects/' + sha1[0,2] + '/' + sha1[2,38]
=> ".git/objects/5e/95f9a59545b492638f77c3832b121e0aeeedbd"
irb(main):009:0> require 'fileutils'
=> true
irb(main):010:0> FileUtils.mkdir_p(File.dirname(path))
=> [".git/objects/5e"]
irb(main):011:0> File.open(path, 'w') { |f| f.write content }
=> 30
```

或者等价于下面的ruby脚本

```ruby
def put_raw_object(content, type)
    size = content.length.to_s
    header = "#{type} #{size}\0" # type(space)size(null byte)
    store = header + content
    sha1 = Digest::SHA1.hexdigest(store)
    path = @git_dir + '/' + sha1[0...2] + '/' + sha1[2..40]
    if !File.exists?(path)
        content = Zlib::Deflate.deflate(store)
        FileUtils.mkdir_p(@directory+'/'+sha1[0...2])
        File.open(path, 'w') do |f|
            f.write content
        end
    end
    return sha1
end
```

### Packfiles

前面所述 Git 默认使用松散对象，当文件内容过大时，少量修改带来的负担就大了，可采用`git gc`增量存储并打包。对比前后可见仅悬空的 object 还保留着，且新增了一个 packfile 及其索引文件。用`git verify-pack`可查看打包文件内容。

```
~/Documents/Github/git-object(branch:master) » find .git/objects -type f
.git/objects/0d/d845d1604b0a93c49d022ac003e0c71e825ff2
.git/objects/40/6b366b64239c963ce752e9b301aaca580c14be
.git/objects/44/0c92d74bae1cce922d2392621086a84839b55f
.git/objects/67/4c6e1d4c29302a1c1f92d6de6dc835b405cf6c
.git/objects/77/e38eb8ed95e8e49a5b8597629d8f00312f4f5a
.git/objects/95/23540db32046e57a16f4408dbdf3a2a972fd32
.git/objects/99/b20f755c8a258cd18a443c8dedfd73c480b24b
.git/objects/c9/c0bfeb224cb32ab143d30a364bec1dd2cc9d95
.git/objects/ed/76f5a420c5c86f27e1adcea9afb4287d1cd49e
.git/objects/f6/1a97dd3270e5f0cd140170bee184c2d2018906
------------------------------------------------------------
~/Documents/Github/git-object(branch:master) » git gc
Counting objects: 8, done.
Delta compression using up to 8 threads.
Compressing objects: 100% (4/4), done.
Writing objects: 100% (8/8), done.
Total 8 (delta 0), reused 0 (delta 0)
------------------------------------------------------------
~/Documents/Github/git-object(branch:master) » find .git/objects -type f
.git/objects/77/e38eb8ed95e8e49a5b8597629d8f00312f4f5a
.git/objects/c9/c0bfeb224cb32ab143d30a364bec1dd2cc9d95
.git/objects/info/packs
.git/objects/pack/pack-35944864af0373bf837883c3770516731dbaad48.idx
.git/objects/pack/pack-35944864af0373bf837883c3770516731dbaad48.pack
------------------------------------------------------------
~/Documents/Github/git-object(branch:master) » git verify-pack -v \
\ .git/objects/pack/pack-35944864af0373bf837883c3770516731dbaad48.idx
406b366b64239c963ce752e9b301aaca580c14be commit 222 150 12
f61a97dd3270e5f0cd140170bee184c2d2018906 commit 174 124 162
674c6e1d4c29302a1c1f92d6de6dc835b405cf6c tag    141 130 286
9523540db32046e57a16f4408dbdf3a2a972fd32 tree   63 73 416
99b20f755c8a258cd18a443c8dedfd73c480b24b tree   40 51 489
440c92d74bae1cce922d2392621086a84839b55f tree   32 43 540
ed76f5a420c5c86f27e1adcea9afb4287d1cd49e blob   13 22 583
0dd845d1604b0a93c49d022ac003e0c71e825ff2 blob   9 18 605
non delta: 8 objects
.git/objects/pack/pack-35944864af0373bf837883c3770516731dbaad48.pack: ok
```

### Refspec

Refspec 的格式是一个可选的 `+` 号，接着是 `src:dst` 的格式。可选的 `+` 号告诉 Git 在即使不能快速演进的情况下，也去强制更新它。这里修改`.git/config`文件为如下（fetch 时 src 是远端仓库，push 时 src 是本地仓库），用命名空间来划分提交，此时默认 push 和 pull 均操作于远端仓库 develop 子目录。

```
~/Documents/Github/git-object(branch:master) » cat .git/config
[remote "origin"]
        url = /Users/suiteki/Documents/Github/git-origin
        fetch = +refs/heads/develop/*:refs/remotes/origin/develop/*
        push = refs/heads/*:refs/heads/develop/*
------------------------------------------------------------
~/Documents/Github/git-object(branch:master) » git push
Total 0 (delta 0), reused 0 (delta 0)
To /Users/suiteki/Documents/Github/git-origin
 * [new branch]      bak -> develop/bak
 * [new branch]      master -> develop/master
------------------------------------------------------------
~/Documents/Github/git-object(branch:master) » git fetch -v
From /Users/suiteki/Documents/Github/git-origin
 = [up to date]      develop/bak -> origin/develop/bak
 = [up to date]      develop/master -> origin/develop/master
```

删除引用，留空 `src` 即可

### 数据恢复

这里先用 `git reset` 恢复到最初的 commit ，此时最快捷的方法是查看 HEAD 的变动日志，用 `git reflog` 或者直接 `cat .git/logs/HEAD` 均可查看到前一个 commit 为 406b366，此时可将其创建新分支或者直接 `git reset` 回来。

```
~/Documents/Github/git-object(branch:master) » git log --pretty=oneline
406b366b64239c963ce752e9b301aaca580c14be version 0.2
f61a97dd3270e5f0cd140170bee184c2d2018906 version 0.1
------------------------------------------------------------
~/Documents/Github/git-object(branch:master) » git reset --hard f61a97d
HEAD is now at f61a97d version 0.1
------------------------------------------------------------
~/Documents/Github/git-object(branch:master) » git reflog
f61a97d HEAD@{0}: reset: moving to f61a97d
406b366 HEAD@{1}: checkout: moving from bak to master
------------------------------------------------------------
~/Documents/Github/git-object(branch:master) » cat .git/logs/HEAD
406b366b64239c963ce752e9b301aaca580c14be f61a97dd3270e5f0cd140170bee184c2d2018906 suiteki <hythyt9898@vip.qq.com> 1412862066 +0800  reset: moving to f61a97d
```

提高难度，假设 commit 和 `.git/logs` 下所有文件均丢失，此时无法用 `git reflog` 查看日志。可用 `git fsck --full` 来检查仓库的数据完整性，将所有未被其他对象引用的悬空对象列出。如果之前用过 `git gc`，该方法也无法找出，只能用 `git verify-pack -v .git/objects/pack-xx.idx` 的方式查找。

```
~/Documents/Github/git-object(branch:bak) » git branch -D master
Deleted branch master (was 406b366).
------------------------------------------------------------
~/Documents/Github/git-object(branch:bak) » rm -rf .git/logs
------------------------------------------------------------
~/Documents/Github/git-object(branch:bak) » git reflog
------------------------------------------------------------
~/Documents/Github/git-object(branch:bak) » git fsck --full
dangling commit 406b366b64239c963ce752e9b301aaca580c14be
```

### 移除对象

如果早期提交中引入了大文件（如库文件之类），即使后面用 `git rm` 将其删除，但该对象依然存在。这里创建先添加一个1MB的文件，修改并提交多次。通过排序找到的最大文件SHA1为 7510e81，用 `git rev-list` 找到对应的文件名为 bigfile，用 `git log` 找到修改过该文件的 commit，可知道最早从 a4b0a2f 开始修改，使用 `git filter-branch` 实现。最后顺便用 `git prune` 将不可达对象都删掉省点空间。

```
~/Documents/Github/git-object(branch:master*) » dd if=/dev/zero of=bigfile bs=1m count=1
1+0 records in
1+0 records out
1048576 bytes transferred in 0.001215 secs (863038954 bytes/sec)
------------------------------------------------------------
~/Documents/Github/git-object(branch:master) » git log --pretty=oneline
21f28b3e127cc6b766996cb36fb87bb674f11a16 add file2
db54398399d4857383bdd3a16b07200647274018 edited bigfile
a4b0a2f5bc3448a6e5a209fad218f6f7cf369051 add bigfile
406b366b64239c963ce752e9b301aaca580c14be version 0.2
f61a97dd3270e5f0cd140170bee184c2d2018906 version 0.1
------------------------------------------------------------
~/Documents/Github/git-object(branch:master) » git verify-pack .git/objects/pack/pack-62b8e9897eaa4bad23c225eb0b2375f6cf405f57.idx -v | sort -k 3 -n
7510e81e388124b2135e0b59e78003d7a036ff6d blob   1048590 1060 1093
------------------------------------------------------------
~/Documents/Github/git-object(branch:master) » git rev-list --objects --all | grep 7510e81
7510e81e388124b2135e0b59e78003d7a036ff6d bigfile
------------------------------------------------------------
~/Documents/Github/git-object(branch:master) » git log --pretty=oneline --branches -- bigfile
db54398399d4857383bdd3a16b07200647274018 edited bigfile
a4b0a2f5bc3448a6e5a209fad218f6f7cf369051 add bigfile
------------------------------------------------------------
~/Documents/Github/git-object(branch:master) » git filter-branch --index-filter 'git rm --cached --ignore-unmatch bigfile' -- a4b0a2f^..
Rewrite a4b0a2f5bc3448a6e5a209fad218f6f7cf369051 (1/3)rm 'bigfile'
Rewrite db54398399d4857383bdd3a16b07200647274018 (2/3)rm 'bigfile'
Rewrite 21f28b3e127cc6b766996cb36fb87bb674f11a16 (3/3)rm 'bigfile'

Ref 'refs/heads/master' was rewritten
------------------------------------------------------------
~/Documents/Github/git-object(branch:master) » git prune
```

关于 `git filter-branch` 的一些说明：
> --index-filter 选项类似于 --tree-filter 选项，但这里不是传入一个命令去修改磁盘上签出的文件，而是修改暂存区域或索引。不能用 rm file 命令来删除一个特定文件，而是必须用 git rm --cached 来删除它 ── 即从索引而不是磁盘删除它。这样做是出于速度考虑 ── 由于 Git 在运行你的 filter 之前无需将所有版本签出到磁盘上，这个操作会快得多。也可以用 --tree-filter 来完成相同的操作。git rm 的 --ignore-unmatch 选项指定当你试图删除的内容并不存在时不显示错误。最后，因为你清楚问题是从哪个 commit 开始的，使用 filter-branch 重写自 6df7640 这个 commit 开始的所有历史记录。不这么做的话会重写所有历史记录，花费不必要的更多时间。

### 参考书目
1. [git community book](http://gitbook.liuhui998.com)
2. [Pro Git](http://git-scm.com/book)
---
title: "PVE虚拟机一次磁盘IO异常的排查解决"
date: 2026-05-31T02:00:00+08:00
description: "PVE虚拟机一次磁盘IO异常的排查解决"
layout: post

tags:
  - Windows
  - GPU直通
  - 磁盘IO
categories:
  - 技术
lightgallery: true

toc:
  auto: true
---

## 1. 背景

   我日常一直是使用 一张 `RTX 2060 Super` 直通的 Windows 11 虚拟机作为我家里日常的主力机器，用来 Coding (ssh Ubuntu)、Gaming (英雄联盟)、以及一些观影（自建 Jellyfin），显示器是小米的 4K，这台虚拟机作为日常机器还是挺稳定的，平时也不用关机，完全能胜任我的日常任务，除了 Windows 本身的原因，长时间不关机会出现一些莫名的 BUG 之外都挺好，直到前段时间我把它玩坏了。

  事情的起因我已经记不清了，是一个极小的问题让我需要更新 Windows，因为我这个版本的 Windows 11 实在太老了，大概是 2022 年的版本 ：22H2，但致命的问题是我的 Windows 是下载的国外大神精简版的，为了极致的优化砍了很多组件，这些精简版为了稳定往往会禁用 Windows 的更新，就怕小白去更新之后黑屏打不开，而我成了那个小白，原因是我在另一台安装了同样系统的物理机上成功更新过一次，那次成功应是侥幸，那次的版本跨度没有很高，可以看作是一次小更新，而这次我直接死在了第一个补丁，安装了这个补丁之后直接黑屏无法启动，毫无疑问需要修复了，精简系统无恢复环境，只能靠 Windows PE 救一救，进了 PE 之后看到这个补丁已经固化，即已经替换掉了系统的一些文件，无法进行恢复，照着 Gemini 给的一些方案试了一下最终还是不行。万幸的是我的代码已经提交到自建 Gitea，我的重要文件是实时 Nextcloud 同步，唯一遗憾的是我做空投任务的时候启动的几个 Chrome 的 Profile 文件夹，里面有一些用来撸毛账号的 Cookie 没有备份，其实吧虚拟机是可以每天备份的，但我为了玩 Steam 把磁盘扩到了 500 G，剩余空间完全不够装下这么大的备份，所以就不了了之了，好在撸毛的钱包是有私钥备份的，况且撸毛次次反撸，我已经不抱希望能领取大毛了，于是连磁盘也懒得拷贝，直接重装算了。



## 2. 重装 Win 11

这次重装我还是选择了 22H2， 不为其他，只因下载了一个新的26H1 的镜像居然安装失败了，所以还是稳定的慢慢升级吧。

下面是我的 Windows 11 虚拟机的硬件图

![PVE 虚拟机硬件](https://img.linkzz.eu.org/main/images/2026/05/ecad246dd85daa9de11712cd49271955.png)
            

初看是没问题的是吧，安装很顺利，也成功升级到了 23H2，但是使用起来总有点怪怪。



## 3. 异常情况

### 3.1 游戏

首先是游戏莫名的卡顿，我这个配置在以前玩 LOL 是没什么问题的，4K 分辨率中画质完全能应付，但是我即使调到低画质还是会莫名的掉帧，CPU 负载打开游戏就开始飙升到百分之八十多

### 3.2 starship

真正让我确认有问题的是 [starship](https://starship.rs/) , 这是一个终端 prompt 工具，用于在终端显示 git 状态，nodejs 版本之类的，内部集成了大量的编程语言运行环境检测，我是用来和 nushell  配合使用，Nushell 是 Rust 开发的跨平台 Shell, 它性能极好，设计优雅，是我在 Windows 下 shell 的不二之选，这次的异常情况就是在任意一个 git 目录下执行 `ls` 命令的时候，starship 的 prompt 总是慢半拍才能显示出来，体感在 500 ms 左右，这是很反常的，因为我在公司的电脑同样的 `ls` 命令 starship 响应体验是很快的，体感在 100 多 ms，我不禁怀疑起了 starship 是否有问题。



## 4. 开始排查

好在 starship 提供详细的日志供调试， 在 Nushell 下设置一下环境变量即可：

```nushell
$env.STARSHIP_LOG = "trace"
```

在任意git目录下的详细日志，已经舍去其他杂音，只留下加载时间：

```log
[TRACE] - (starship::modules): Took 18.3062ms to compute module "pijul_channel"
[TRACE] - (starship::modules): Took 17.505ms to compute module "nats"
[TRACE] - (starship::modules): Took 17.746ms to compute module "cpp"
[TRACE] - (starship::modules): Took 17.4448ms to compute module "hg_branch"
[TRACE] - (starship::modules): Took 33.8159ms to compute module "odin"
[TRACE] - (starship::modules): Took 17.7322ms to compute module "username"
[TRACE] - (starship::modules): Took 14.7262ms to compute module "pixi"
[TRACE] - (starship::modules): Took 15.0693ms to compute module "git_metrics"
[TRACE] - (starship::modules): Took 1.4938ms to compute module "hg_state"
[TRACE] - (starship::modules): Took 17.8735ms to compute module "docker_context"
[TRACE] - (starship::modules): Took 18.5529ms to compute module "hostname"
[TRACE] - (starship::modules): Took 16.166ms to compute module "opa"
[TRACE] - (starship::modules): Took 15.623ms to compute module "daml"
[TRACE] - (starship::modules): Took 16.2233ms to compute module "fossil_branch"
[TRACE] - (starship::modules): Took 15.1224ms to compute module "localip"
[TRACE] - (starship::modules): Took 2.3825ms to compute module "perl"
[TRACE] - (starship::modules): Took 15.1989ms to compute module "bun"
[TRACE] - (starship::modules): Took 29.0678ms to compute module "fossil_metrics"
[TRACE] - (starship::modules): Took 17.8275ms to compute module "rust"
[TRACE] - (starship::modules): Took 16.9605ms to compute module "shlvl"
[TRACE] - (starship::modules): Took 17.2842ms to compute module "python"
[TRACE] - (starship::modules): Took 16.9597ms to compute module "memory_usage"
[TRACE] - (starship::modules): Took 47.9592ms to compute module "dart"
[TRACE] - (starship::modules): Took 18.5844ms to compute module "scala"
[TRACE] - (starship::modules): Took 19.0209ms to compute module "php"
[TRACE] - (starship::modules): Took 20.7344ms to compute module "c"
[TRACE] - (starship::modules): Took 16.4049ms to compute module "quarto"
[TRACE] - (starship::modules): Took 2.4443ms to compute module "kubernetes"
[TRACE] - (starship::modules): Took 20.0147ms to compute module "deno"
[TRACE] - (starship::modules): Took 18.8966ms to compute module "solidity"
[TRACE] - (starship::modules): Took 17.4583ms to compute module "cmake"
[TRACE] - (starship::modules): Took 17.3312ms to compute module "openstack"
[TRACE] - (starship::modules): Took 15.9276ms to compute module "purescript"
[TRACE] - (starship::modules): Took 16.5826ms to compute module "raku"
[TRACE] - (starship::modules): Took 18.8865ms to compute module "gradle"
[TRACE] - (starship::modules): Took 17.4403ms to compute module "dotnet"
[TRACE] - (starship::modules): Took 16.2543ms to compute module "swift"
[TRACE] - (starship::modules): Took 18.1222ms to compute module "azure"
[TRACE] - (starship::modules): Took 16.8151ms to compute module "cobol"
[TRACE] - (starship::modules): Took 33.1375ms to compute module "rlang"
[TRACE] - (starship::modules): Took 17.1506ms to compute module "elm"
[TRACE] - (starship::modules): Took 18.019ms to compute module "haskell"
[TRACE] - (starship::modules): Took 16.4837ms to compute module "terraform"
[TRACE] - (starship::modules): Took 25.0324ms to compute module "elixir"
[TRACE] - (starship::modules): Took 16.7054ms to compute module "direnv"
[TRACE] - (starship::modules): Took 19.2445ms to compute module "sudo"
[TRACE] - (starship::modules): Took 20.1962ms to compute module "red"
[TRACE] - (starship::modules): Took 280.4748ms to compute module "git_branch"
[TRACE] - (starship::modules): Took 19.7518ms to compute module "erlang"
[TRACE] - (starship::modules): Took 19.5443ms to compute module "haxe"
[TRACE] - (starship::modules): Took 18.5698ms to compute module "typst"
[TRACE] - (starship::modules): Took 17.3636ms to compute module "env_var"
[TRACE] - (starship::modules): Took 16.9152ms to compute module "kotlin"
[TRACE] - (starship::modules): Took 2.5295ms to compute module "vlang"
[TRACE] - (starship::modules): Took 19.6208ms to compute module "ruby"
[TRACE] - (starship::modules): Took 18.6619ms to compute module "zig"
[TRACE] - (starship::modules): Took 36.1428ms to compute module "cmd_duration"
[TRACE] - (starship::modules): Took 19.5368ms to compute module "helm"
[TRACE] - (starship::modules): Took 17.5727ms to compute module "vagrant"
[TRACE] - (starship::modules): Took 18.321ms to compute module "mise"
[TRACE] - (starship::modules): Took 33.962ms to compute module "fennel"
[TRACE] - (starship::modules): Took 17.4341ms to compute module "lua"
[TRACE] - (starship::modules): Took 1.585ms to compute module "line_break"
[TRACE] - (starship::modules): Took 17.5436ms to compute module "crystal"
[TRACE] - (starship::modules): Took 20.7769ms to compute module "java"
[TRACE] - (starship::modules): Took 19.8726ms to compute module "buf"
[TRACE] - (starship::modules): Took 19.44ms to compute module "fortran"
[TRACE] - (starship::modules): Took 19.7548ms to compute module "guix_shell"
[TRACE] - (starship::modules): Took 17.8847ms to compute module "xmake"
[TRACE] - (starship::modules): Took 17.9972ms to compute module "jobs"
[TRACE] - (starship::modules): Took 17.7127ms to compute module "maven"
[TRACE] - (starship::modules): Took 2.4656ms to compute module "julia"
[TRACE] - (starship::modules): Took 26.8797ms to compute module "mojo"
[TRACE] - (starship::modules): Took 21.4463ms to compute module "gleam"
[TRACE] - (starship::modules): Took 2.9202ms to compute module "battery"
[TRACE] - (starship::modules): Took 20.1832ms to compute module "nix_shell"
[TRACE] - (starship::modules): Took 23.8637ms to compute module "nim"
[TRACE] - (starship::modules): Took 16.0568ms to compute module "time"
[TRACE] - (starship::modules): Took 16.825ms to compute module "golang"
[TRACE] - (starship::modules): Took 32.403ms to compute module "ocaml"
[TRACE] - (starship::modules): Took 85.444ms to compute module "shell"
[TRACE] - (starship::modules): Took 80.4897ms to compute module "status"
[TRACE] - (starship::modules): Took 83.2728ms to compute module "os"
[TRACE] - (starship::modules): Took 159.0895ms to compute module "character"
```

同样在公司机器上也测一测

```log
➜ open tmp\starship-1.log
[TRACE] - (starship::modules): Took 5.7994ms to compute module "vlang"
[TRACE] - (starship::modules): Took 7.4286ms to compute module "username"
[TRACE] - (starship::modules): Took 4.3847ms to compute module "rust"
[TRACE] - (starship::modules): Took 8.69ms to compute module "opa"
[TRACE] - (starship::modules): Took 5.5902ms to compute module "guix_shell"
[TRACE] - (starship::modules): Took 16.9084ms to compute module "hostname"
[TRACE] - (starship::modules): Took 5.6574ms to compute module "php"
[TRACE] - (starship::modules): Took 4.5768ms to compute module "memory_usage"
[TRACE] - (starship::modules): Took 1.0898ms to compute module "cobol"
[TRACE] - (starship::modules): Took 17.9987ms to compute module "python"
[TRACE] - (starship::modules): Took 6.7587ms to compute module "localip"
[TRACE] - (starship::modules): Took 6.7914ms to compute module "swift"
[TRACE] - (starship::modules): Took 5.7805ms to compute module "terraform"
[TRACE] - (starship::modules): Took 5.4692ms to compute module "daml"
[TRACE] - (starship::modules): Took 10.6374ms to compute module "shlvl"
[TRACE] - (starship::modules): Took 5.5323ms to compute module "raku"
[TRACE] - (starship::modules): Took 15.6917ms to compute module "gradle"
[TRACE] - (starship::modules): Took 34.5668ms to compute module "gcloud"
[TRACE] - (starship::modules): Took 49.7071ms to compute module "git_branch"
[TRACE] - (starship::modules): Took 6.4357ms to compute module "dart"
[TRACE] - (starship::modules): Took 6.1071ms to compute module "openstack"
[TRACE] - (starship::modules): Took 5.1708ms to compute module "erlang"
[TRACE] - (starship::modules): Took 4.9006ms to compute module "kubernetes"
[TRACE] - (starship::modules): Took 16.7163ms to compute module "rlang"
[TRACE] - (starship::modules): Took 5.912ms to compute module "direnv"
[TRACE] - (starship::modules): Took 13.8326ms to compute module "helm"
[TRACE] - (starship::modules): Took 6.0494ms to compute module "dotnet"
[TRACE] - (starship::modules): Took 85.5848ms to compute module "pulumi"
[TRACE] - (starship::modules): Took 9.669ms to compute module "kotlin"
[TRACE] - (starship::modules): Took 6.7169ms to compute module "time"
[TRACE] - (starship::modules): Took 7.5859ms to compute module "cmd_duration"
[TRACE] - (starship::modules): Took 6.7803ms to compute module "lua"
[TRACE] - (starship::modules): Took 13.5457ms to compute module "git_commit"
[TRACE] - (starship::modules): Took 17.2346ms to compute module "deno"
[TRACE] - (starship::modules): Took 9.7572ms to compute module "line_break"
[TRACE] - (starship::modules): Took 33.0256ms to compute module "directory"
[TRACE] - (starship::modules): Took 4.2482ms to compute module "fossil_branch"
[TRACE] - (starship::modules): Took 4.302ms to compute module "nim"
[TRACE] - (starship::modules): Took 1.0841ms to compute module "fossil_metrics"
[TRACE] - (starship::modules): Took 5.3851ms to compute module "nodejs"
[TRACE] - (starship::modules): Took 36.5341ms to compute module "red"
[TRACE] - (starship::modules): Took 5.3772ms to compute module "os"
[TRACE] - (starship::modules): Took 5.3431ms to compute module "jobs"
[TRACE] - (starship::modules): Took 6.672ms to compute module "ocaml"
[TRACE] - (starship::modules): Took 5.2113ms to compute module "git_metrics"
[TRACE] - (starship::modules): Took 5.0682ms to compute module "git_state"
[TRACE] - (starship::modules): Took 5.3172ms to compute module "odin"
[TRACE] - (starship::modules): Took 6.8135ms to compute module "shell"
[TRACE] - (starship::modules): Took 6.7616ms to compute module "hg_branch"
[TRACE] - (starship::modules): Took 25.748ms to compute module "battery"
[TRACE] - (starship::modules): Took 21.8025ms to compute module "character"
[TRACE] - (starship::utils): stdout: "# branch.oid 2e0893e49032c5d0cfb2a56f720d7356f7e23685\n# branch.head main\n# branch.upstream origin/main\n# branch.ab +0 -0\n", stderr: "", exit code: "Some(0)", took 50.7518ms
[TRACE] - (starship::modules): Took 99.5484ms to compute module "git_status"
```

我发现公司机器上的加载时间都很小，尤其是稍微耗时的 git 相关模块，联想到 git 需要读取文件，涉及到磁盘 IO，于是我打开 Gemini 询问了磁盘测试的工具，Gemini 推荐了一个微软官方发布的[小工具](https://github.com/microsoft/diskspd/releases)用来测试:

```shellscript
diskspd -c1G -d60 -r -w40 -t4 -o32 -b4K -Sh -L c:\testfile.dat
```

测试结果：

```log

Input parameters:

        timespan:   1
        -------------
        duration: 60s
        warm up time: 5s
        cool down time: 0s
        measuring latency
        random seed: 0
        path: 'c:\testfile.dat'
                think time: 0ms
                burst size: 0
                software cache disabled
                hardware write cache disabled, writethrough on
                performing mix test (read/write ratio: 60/40)
                block size: 4KiB
                using random I/O (alignment: 4KiB)
                number of outstanding I/O operations per thread: 32
                threads per file: 4
                using I/O Completion Ports
                IO priority: normal

System information:

        computer name: DESKTOP-61E3A0U
        start time: 2026/05/30 04:38:55 UTC

        cpu count:              10
        core count:             10
        group count:            1
        node count:             1
        socket count:           1
        heterogeneous cores:    n

        active power scheme:    骞宠　 (381b4222-f694-41f0-9685-ff5bb260df2e)

Results for timespan 1:
*******************************************************************************

actual test time:       60.01s
thread count:           4

CPU |  Usage |  User  | Kernel |  Idle
----------------------------------------
   0|  94.14%|   5.78%|  88.36%|   5.86%
   1|  98.49%|   7.89%|  90.60%|   1.51%
   2|  94.69%|   5.02%|  89.66%|   5.31%
   3|  94.04%|   4.76%|  89.27%|   5.96%
   4|  43.72%|  25.86%|  17.86%|  56.28%
   5|  37.18%|  21.37%|  15.80%|  62.82%
   6|  37.10%|  19.08%|  18.02%|  62.90%
   7|  31.87%|  18.56%|  13.30%|  68.13%
   8|  95.49%|  11.09%|  84.40%|   4.51%
   9|  37.92%|  19.32%|  18.59%|  62.08%
----------------------------------------
avg.|  66.46%|  13.88%|  52.59%|  33.54%

Total IO
thread |       bytes     |     I/Os     |    MiB/s   |  I/O per s |  AvgLat  | LatStdDev |  file
-----------------------------------------------------------------------------------------------------
     0 |      1582923776 |       386456 |      25.16 |    6440.19 |    1.162 |     2.720 | c:\testfile.dat (1GiB)
     1 |      1581031424 |       385994 |      25.13 |    6432.50 |    1.171 |     2.791 | c:\testfile.dat (1GiB)
     2 |      1667600384 |       407129 |      26.50 |    6784.70 |    1.126 |     2.753 | c:\testfile.dat (1GiB)
     3 |      1688813568 |       412308 |      26.84 |    6871.01 |    1.112 |     2.748 | c:\testfile.dat (1GiB)
-----------------------------------------------------------------------------------------------------
total:        6520369152 |      1591887 |     103.63 |   26528.41 |    1.142 |     2.753

Read IO
thread |       bytes     |     I/Os     |    MiB/s   |  I/O per s |  AvgLat  | LatStdDev |  file
-----------------------------------------------------------------------------------------------------
     0 |       950878208 |       232148 |      15.11 |    3868.69 |    1.224 |     2.730 | c:\testfile.dat (1GiB)
     1 |       950878208 |       232148 |      15.11 |    3868.69 |    1.238 |     2.849 | c:\testfile.dat (1GiB)
     2 |       999952384 |       244129 |      15.89 |    4068.35 |    1.186 |     2.764 | c:\testfile.dat (1GiB)
     3 |      1013227520 |       247370 |      16.10 |    4122.36 |    1.176 |     2.792 | c:\testfile.dat (1GiB)
-----------------------------------------------------------------------------------------------------
total:        3914936320 |       955795 |      62.22 |   15928.09 |    1.206 |     2.784

Write IO
thread |       bytes     |     I/Os     |    MiB/s   |  I/O per s |  AvgLat  | LatStdDev |  file
-----------------------------------------------------------------------------------------------------
     0 |       632045568 |       154308 |      10.04 |    2571.50 |    1.068 |     2.702 | c:\testfile.dat (1GiB)
     1 |       630153216 |       153846 |      10.01 |    2563.81 |    1.069 |     2.698 | c:\testfile.dat (1GiB)
     2 |       667648000 |       163000 |      10.61 |    2716.35 |    1.036 |     2.734 | c:\testfile.dat (1GiB)
     3 |       675586048 |       164938 |      10.74 |    2748.65 |    1.015 |     2.678 | c:\testfile.dat (1GiB)
-----------------------------------------------------------------------------------------------------
total:        2605432832 |       636092 |      41.41 |   10600.32 |    1.046 |     2.703

Total latency distribution:
  %-ile |  Read (ms) | Write (ms) | Total (ms)
----------------------------------------------
    min |      0.123 |      0.083 |      0.083
   25th |      0.283 |      0.194 |      0.242
   50th |      0.350 |      0.245 |      0.311
   75th |      0.536 |      0.336 |      0.456
   90th |      3.958 |      3.919 |      3.942
   95th |      4.635 |      4.592 |      4.617
   99th |     12.981 |     12.781 |     12.896
3-nines |     36.063 |     33.409 |     35.402
4-nines |     69.724 |     66.539 |     68.683
5-nines |     72.714 |     72.588 |     72.685
6-nines |     74.241 |     74.249 |     74.241
7-nines |     74.241 |     74.249 |     74.249
8-nines |     74.241 |     74.249 |     74.249
9-nines |     74.241 |     74.249 |     74.249
    max |     74.241 |     74.249 |     74.249
```

把结果让 Gemini 分析就一目了然了，其实我在测试的时候看资源管理器就已经发现了端倪，我测磁盘 IO 最高只能跑到 100M/s，即使是虚拟机有损耗这个结果也是很反常的，果然 Gemini 给出了诊断并给出了解决方案，原来是 **虚拟磁盘使用Sata 总线的原因，不过话说回来我选择 Sata 总线这也是 Gemini 给的建议 🤣**


![磁盘IO测试](https://img.linkzz.eu.org/main/images/2026/05/e699da94895e7a90c341a8c8d031dc73.png)

## 5. 异常解决

既然已经有了解决方案，那我们就按照以上的步骤来解决吧，解决的时候有个坑，我第一张图的 SCSI 控制器是 Virtio SCSI，这个时候添加一块 SCSI 总线的硬盘是无效的，还是会添加成 SATA 总线的盘，因为只有 VirtIO SCSI single 控制器才支持多个总线，所以第一步添加 SCSI 盘让 windows 装驱动的这步我踩了坑，导致更改了系统盘为 SCSI 之后蓝屏了，后续知道了之后总算成功启动，下面再来测一测问题：

### 5.1 starship

首先是 starship, 同样测试一下：

```log
[TRACE] - (starship::modules): Took 21.2983ms to compute module "odin"
[TRACE] - (starship::modules): Took 19.1956ms to compute module "python"
[TRACE] - (starship::modules): Took 31.5634ms to compute module "rlang"
[TRACE] - (starship::modules): Took 36.4878ms to compute module "username"
[TRACE] - (starship::modules): Took 23.9048ms to compute module "rust"
[TRACE] - (starship::modules): Took 16.5662ms to compute module "cpp"
[TRACE] - (starship::modules): Took 18.1202ms to compute module "pixi"
[TRACE] - (starship::modules): Took 17.81ms to compute module "hostname"
[TRACE] - (starship::modules): Took 17.3318ms to compute module "red"
[TRACE] - (starship::modules): Took 17.6097ms to compute module "opa"
[TRACE] - (starship::modules): Took 17.439ms to compute module "quarto"
[TRACE] - (starship::modules): Took 13.4106ms to compute module "daml"
[TRACE] - (starship::modules): Took 17.8897ms to compute module "scala"
[TRACE] - (starship::modules): Took 17.2203ms to compute module "raku"
[TRACE] - (starship::modules): Took 15.8997ms to compute module "perl"
[TRACE] - (starship::modules): Took 16.2383ms to compute module "shlvl"
[TRACE] - (starship::modules): Took 14.9016ms to compute module "dart"
[TRACE] - (starship::modules): Took 15.3076ms to compute module "solidity"
[TRACE] - (starship::modules): Took 30.2241ms to compute module "ruby"
[TRACE] - (starship::modules): Took 19.204ms to compute module "memory_usage"
[TRACE] - (starship::modules): Took 2.5528ms to compute module "swift"
[TRACE] - (starship::modules): Took 16.6836ms to compute module "php"
[TRACE] - (starship::modules): Took 15.8251ms to compute module "pijul_channel"
[TRACE] - (starship::modules): Took 18.0944ms to compute module "kubernetes"
[TRACE] - (starship::modules): Took 17.6829ms to compute module "deno"
[TRACE] - (starship::modules): Took 268.2773ms to compute module "git_commit"
[TRACE] - (starship::modules): Took 2.1399ms to compute module "nats"
[TRACE] - (starship::modules): Took 2.0977ms to compute module "vlang"
[TRACE] - (starship::modules): Took 17.9548ms to compute module "docker_context"
[TRACE] - (starship::modules): Took 18.3204ms to compute module "purescript"
[TRACE] - (starship::modules): Took 2.4789ms to compute module "terraform"
[TRACE] - (starship::modules): Took 47.3202ms to compute module "openstack"
[TRACE] - (starship::modules): Took 16.6641ms to compute module "git_state"
[TRACE] - (starship::modules): Took 16.8515ms to compute module "vagrant"
[TRACE] - (starship::modules): Took 47.4419ms to compute module "dotnet"
[TRACE] - (starship::modules): Took 19.1804ms to compute module "fossil_branch"
[TRACE] - (starship::modules): Took 16.8351ms to compute module "typst"
[TRACE] - (starship::modules): Took 32.1791ms to compute module "bun"
[TRACE] - (starship::modules): Took 18.3454ms to compute module "azure"
[TRACE] - (starship::modules): Took 19.6604ms to compute module "elixir"
[TRACE] - (starship::modules): Took 24.8527ms to compute module "git_metrics"
[TRACE] - (starship::modules): Took 13.2511ms to compute module "xmake"
[TRACE] - (starship::modules): Took 106.3242ms to compute module "directory"
[TRACE] - (starship::modules): Took 14.7182ms to compute module "c"
[TRACE] - (starship::modules): Took 1.186ms to compute module "fossil_metrics"
[TRACE] - (starship::modules): Took 13.7763ms to compute module "elm"
[TRACE] - (starship::modules): Took 14.3267ms to compute module "zig"
[TRACE] - (starship::modules): Took 14.532ms to compute module "gradle"
[TRACE] - (starship::modules): Took 14.8623ms to compute module "direnv"
[TRACE] - (starship::modules): Took 14.1082ms to compute module "sudo"
[TRACE] - (starship::modules): Took 16.1944ms to compute module "env_var"
[TRACE] - (starship::modules): Took 17.6322ms to compute module "buf"
[TRACE] - (starship::modules): Took 17.1988ms to compute module "erlang"
[TRACE] - (starship::modules): Took 15.81ms to compute module "haskell"
[TRACE] - (starship::modules): Took 15.7842ms to compute module "cmake"
[TRACE] - (starship::modules): Took 39.3232ms to compute module "cmd_duration"
[TRACE] - (starship::modules): Took 113.3744ms to compute module "git_branch"
[TRACE] - (starship::modules): Took 39.2447ms to compute module "fennel"
[TRACE] - (starship::modules): Took 1.1245ms to compute module "mise"
[TRACE] - (starship::modules): Took 13.4523ms to compute module "haxe"
[TRACE] - (starship::modules): Took 13.5966ms to compute module "cobol"
[TRACE] - (starship::utils): stdout: "refs/remotes/origin/main \n", stderr: "", exit code: "Some(0)", took 42.702ms
[TRACE] - (starship::modules): Took 12.788ms to compute module "nix_shell"
[TRACE] - (starship::modules): Took 26.5448ms to compute module "guix_shell"
[TRACE] - (starship::modules): Took 12.8141ms to compute module "helm"
[TRACE] - (starship::modules): Took 207.7285ms to compute module "git_status"
[TRACE] - (starship::modules): Took 15.8917ms to compute module "line_break"
[TRACE] - (starship::modules): Took 12.197ms to compute module "time"
[TRACE] - (starship::modules): Took 25.5358ms to compute module "fortran"
[TRACE] - (starship::modules): Took 1.1664ms to compute module "hg_state"
[TRACE] - (starship::modules): Took 2.4666ms to compute module "battery"
[TRACE] - (starship::modules): Took 38.6196ms to compute module "crystal"
[TRACE] - (starship::modules): Took 18.8832ms to compute module "hg_branch"
[TRACE] - (starship::modules): Took 11.4009ms to compute module "jobs"
[TRACE] - (starship::modules): Took 17.8967ms to compute module "java"
[TRACE] - (starship::modules): Took 19.9104ms to compute module "gleam"
[TRACE] - (starship::modules): Took 29.7799ms to compute module "status"
[TRACE] - (starship::modules): Took 16.7913ms to compute module "kotlin"
[TRACE] - (starship::modules): Took 17.0952ms to compute module "mojo"
[TRACE] - (starship::modules): Took 15.6663ms to compute module "ocaml"
[TRACE] - (starship::modules): Took 28.9999ms to compute module "nim"
[TRACE] - (starship::modules): Took 1.222ms to compute module "shell"
[TRACE] - (starship::modules): Took 29.4475ms to compute module "julia"
[TRACE] - (starship::modules): Took 75.4068ms to compute module "maven"
[TRACE] - (starship::modules): Took 64.2614ms to compute module "golang"
[TRACE] - (starship::modules): Took 18.5331ms to compute module "character"
[TRACE] - (starship::modules): Took 73.2697ms to compute module "os"
[TRACE] - (starship::modules): Took 134.8461ms to compute module "lua"
```

可以看到 `git_branch` 从 280ms 降到了 113ms， 使用 ls 命令也顺畅多了

### 5.2 再测磁盘IO



```shellscript
diskspd -c1G -d60 -r -w40 -t4 -o32 -b4K -Sh -L c:\testfile.dat
```

```log
➜ diskspd -c1G -d60 -r -w40 -t4 -o32 -b4K -Sh -L c:\testfile.dat
WARNING: Error adjusting token privileges for SeManageVolumePrivilege (error code: 1300)
WARNING: Could not set privileges for setting valid file size; will use a slower method of preparing the file

Command Line: C:\Users\linkzz\Downloads\diskspd.exe -c1G -d60 -r -w40 -t4 -o32 -b4K -Sh -L c:\testfile.dat

Input parameters:

        timespan:   1
        -------------
        duration: 60s
        warm up time: 5s
        cool down time: 0s
        measuring latency
        random seed: 0
        path: 'c:\testfile.dat'
                think time: 0ms
                burst size: 0
                software cache disabled
                hardware write cache disabled, writethrough on
                performing mix test (read/write ratio: 60/40)
                block size: 4KiB
                using random I/O (alignment: 4KiB)
                number of outstanding I/O operations per thread: 32
                threads per file: 4
                using I/O Completion Ports
                IO priority: normal

System information:

        computer name: DESKTOP-61E3A0U
        start time: 2026/05/30 09:35:27 UTC

        cpu count:              10
        core count:             10
        group count:            1
        node count:             1
        socket count:           1
        heterogeneous cores:    n

        active power scheme:    骞宠　 (381b4222-f694-41f0-9685-ff5bb260df2e)

Results for timespan 1:
*******************************************************************************

actual test time:       60.00s
thread count:           4

CPU |  Usage |  User  | Kernel |  Idle
----------------------------------------
   0|  29.77%|   3.18%|  26.59%|  70.23%
   1|  29.69%|   7.76%|  21.93%|  70.31%
   2|  22.19%|   4.45%|  17.73%|  77.81%
   3|  22.29%|   3.98%|  18.31%|  77.71%
   4|   6.51%|   2.79%|   3.72%|  93.49%
   5|   3.72%|   1.82%|   1.90%|  96.28%
   6|   3.78%|   2.40%|   1.38%|  96.22%
   7|  13.52%|  11.30%|   2.21%|  86.48%
   8|   6.64%|   5.18%|   1.46%|  93.36%
   9|  12.06%|   4.30%|   7.76%|  87.94%
----------------------------------------
avg.|  15.02%|   4.72%|  10.30%|  84.98%

Total IO
thread |       bytes     |     I/Os     |    MiB/s   |  I/O per s |  AvgLat  | LatStdDev |  file
-----------------------------------------------------------------------------------------------------
     0 |      2024943616 |       494371 |      32.19 |    8239.51 |    3.651 |     6.070 | c:\testfile.dat (1GiB)
     1 |      2005823488 |       489703 |      31.88 |    8161.71 |    3.702 |     6.360 | c:\testfile.dat (1GiB)
     2 |      2056142848 |       501988 |      32.68 |    8366.46 |    3.603 |     6.033 | c:\testfile.dat (1GiB)
     3 |      2048163840 |       500040 |      32.55 |    8334.00 |    3.623 |     6.058 | c:\testfile.dat (1GiB)
-----------------------------------------------------------------------------------------------------
total:        8135073792 |      1986102 |     129.30 |   33101.69 |    3.644 |     6.131

Read IO
thread |       bytes     |     I/Os     |    MiB/s   |  I/O per s |  AvgLat  | LatStdDev |  file
-----------------------------------------------------------------------------------------------------
     0 |      1216397312 |       296972 |      19.33 |    4949.53 |    3.796 |     6.039 | c:\testfile.dat (1GiB)
     1 |      1203802112 |       293897 |      19.13 |    4898.28 |    3.844 |     6.344 | c:\testfile.dat (1GiB)
     2 |      1233788928 |       301218 |      19.61 |    5020.30 |    3.752 |     6.010 | c:\testfile.dat (1GiB)
     3 |      1228251136 |       299866 |      19.52 |    4997.76 |    3.755 |     6.019 | c:\testfile.dat (1GiB)
-----------------------------------------------------------------------------------------------------
total:        4882239488 |      1191953 |      77.60 |   19865.87 |    3.786 |     6.104

Write IO
thread |       bytes     |     I/Os     |    MiB/s   |  I/O per s |  AvgLat  | LatStdDev |  file
-----------------------------------------------------------------------------------------------------
     0 |       808546304 |       197399 |      12.85 |    3289.98 |    3.433 |     6.109 | c:\testfile.dat (1GiB)
     1 |       802021376 |       195806 |      12.75 |    3263.43 |    3.488 |     6.379 | c:\testfile.dat (1GiB)
     2 |       822353920 |       200770 |      13.07 |    3346.17 |    3.381 |     6.060 | c:\testfile.dat (1GiB)
     3 |       819912704 |       200174 |      13.03 |    3336.23 |    3.426 |     6.110 | c:\testfile.dat (1GiB)
-----------------------------------------------------------------------------------------------------
total:        3252834304 |       794149 |      51.70 |   13235.81 |    3.432 |     6.165

Total latency distribution:
  %-ile |  Read (ms) | Write (ms) | Total (ms)
----------------------------------------------
    min |      0.044 |      0.037 |      0.037
   25th |      0.395 |      0.160 |      0.271
   50th |      0.917 |      0.360 |      0.638
   75th |      3.324 |      3.469 |      3.381
   90th |     12.617 |     12.343 |     12.511
   95th |     15.871 |     15.608 |     15.768
   99th |     27.832 |     27.690 |     27.774
3-nines |     42.772 |     42.082 |     42.618
4-nines |     66.620 |     66.518 |     66.607
5-nines |     77.884 |     78.000 |     77.919
6-nines |     78.197 |     78.211 |     78.211
7-nines |     78.263 |     78.211 |     78.263
8-nines |     78.263 |     78.211 |     78.263
9-nines |     78.263 |     78.211 |     78.263
    max |     78.263 |     78.211 |     78.263
```

可以看到IOPS 26528.41 提升到了 33101.69，吞吐量也从 103.63 MiB/s 提升到了 129.30 MiB/s，最主要的是 CPU 占用不再飙升，至此问题算是解决了。

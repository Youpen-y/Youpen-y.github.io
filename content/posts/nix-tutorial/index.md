---
title: "Nix Tutorial"
date: 2026-06-14T21:34:56+08:00
draft: false
tags: ["nix"]
comments: true
---
AI 代理的沙盒开发环境是当前研究的一个热点方向，前几天看到了 Ubuntu 发布了 [workshop](https://ubuntu.com/workshop)，觉得很有趣，但是还没来得及细细研究。由于工作环境目前迁移到了 Arch，便选择 Nix 先做一下这方面的入门工作。

Nix 作为一个软件部署系统，由 Eelco Dolstra 等人于 2004 年发明，论文可在 [Nix: A Safe and Policy-Free System for Software Deployment](https://www.usenix.org/legacy/publications/library/proceedings/lisa04/tech/full_papers/dolstra/dolstra_html/index.html) 处查看。主要用于解决当时传统软件部署的两大类问题。

第一类问题是安全性问题：
- 没有执行可靠的软件依赖规范：实际上就是常见的“程序在我的机器上可以运行”问题。一个程序往往会依赖很多组件库，组件库的有无和版本直接影响程序能否正常运行。有些软件包管理器没有正确识别组件的依赖关系，还有一些则识别不完整。
- 单个系统不支持存在多个版本的组件库：升级组件库往往使用覆盖的方式，无法同时安装组件库的多个版本。
- 软件升级非原子操作：覆盖式升级存在时间窗口，在时间窗口内，组件库处于一种中间态（半新半旧），很可能导致依赖它的软件无法正常工作。
- 标识问题：多个组件源可能包含同名组件，但其内容却不相同。

第二类问题是灵活性问题：
- 源码部署和可执行文件部署的割裂：传统方案需要为一个组件创建源码包和可执行文件包，但可执行文件实际上是源码部署的优化，理想情况下，从源码定义到可执行文件应该是自动、透明地完成，而不用手动维护两套东西。
- 软件部署应既支持集中式包管理，也支持本地包管理：一些软件服务在同一网络下只需部署一次，而另一些用户个性化软件服务需要本地软件包管理支持。

### Nix 的核心方案
Nix 采用了一种非常直观的技术来解决上述问题：对构建的全部输入（依赖）做加密哈希，从生成唯一的存储路径。

主要包含以下组成部分：
1. `Nix Store`（存储区）。\
存储区用于存放 Nix 管理的应用程序及其所有依赖组件库，并使用目录来区分不同组件（`/nix/store/<32位hash>-<名字>`）。通过哈希来唯一标识组件，使得多个版本的组件可以共存与存储区而不会冲突。

2. `Nix Expressions`（Nix 表达式）。\
一种声明式函数语言，描述组件如何从源码构建、依赖的组件库，以及对依赖项的约束。Nix 表达式经过求值后，会生成 derivation（构建配方），这是一种确定的、不可变的构建描述，记录了所有输入依赖、构建脚本和输出路径，存储为 `/nix/store` 中的 `.drv` 文件。derivation 的哈希由其全部输入决定，相同的哈希必然产生相同的构建产物，这是构建共享的基础。

示例：
```nix
# hello.nix
{ pkgs ? import <nixpkgs> {} }:
pkgs.stdenv.mkDerivation {
  name = "hello-2.12.1";
  src = pkgs.fetchurl {
    url = "https://ftp.gnu.org/gnu/hello/hello-2.12.1.tar.gz";
    hash = "sha256-jZkUKv2SV28wsM18tCqNxoCZmLxdYH2Idh9RLibH2yA=";
  };
  buildInputs = [ pkgs.gcc pkgs.gnumake ];
  buildPhase = ''
    ./configure
    make
  '';
  installPhase = ''
    make install DESTDIR=$out
  '';
}
```
翻译为 derivation ：
```bash
$ nix-instantiate hello.nix
/nix/store/wl23pfcc0ps9z58iipcg4hmkdz3x69qx-hello-2.12.1.drv
```
同样，也可以打开 `.drv` 查看其内容
```bash
nix show-derivation /nix/store/wl23pfcc0ps9z58iipcg4hmkdz3x69qx-hello-2.12.1.drv
```
```json
{
  "derivations": {
    "wl23pfcc0ps9z58iipcg4hmkdz3x69qx-hello-2.12.1.drv": {
      "args": [
        "-e",
        "/nix/store/l622p70vy8k5sh7y5wizi5f2mic6ynpg-source-stdenv.sh",
        "/nix/store/shkw4qm9qcw5sc5n1k5jznc83ny02r39-default-builder.sh"
      ],
      "builder": "/nix/store/gik3rh1vz2jlgnifb9dh6vc6sxwwz9jj-bash-5.3p9/bin/bash",
      "env": {
        "__structuredAttrs": "",
        "buildInputs": "/nix/store/788mx070y81zjlg5ipcl0cra3afviw9k-gcc-wrapper-15.2.0 /nix/store/d3bwqm6bymhy3pdgbvf7vxjqfp31m3j1-gnumake-4.4.1",
        "buildPhase": "./configure\nmake\n",
        "builder": "/nix/store/gik3rh1vz2jlgnifb9dh6vc6sxwwz9jj-bash-5.3p9/bin/bash",
        "cmakeFlags": "",
        "configureFlags": "",
        "depsBuildBuild": "",
        "depsBuildBuildPropagated": "",
        "depsBuildTarget": "",
        "depsBuildTargetPropagated": "",
        "depsHostHost": "",
        "depsHostHostPropagated": "",
        "depsTargetTarget": "",
        "depsTargetTargetPropagated": "",
        "doCheck": "",
        "doInstallCheck": "",
        "installPhase": "make install DESTDIR=$out\n",
        "mesonFlags": "",
        "name": "hello-2.12.1",
        "nativeBuildInputs": "",
        "out": "/nix/store/rc0j253ssqp88v8xzy5710ndz09kxnwy-hello-2.12.1",
        "outputs": "out",
        "patches": "",
        "propagatedBuildInputs": "",
        "propagatedNativeBuildInputs": "",
        "src": "/nix/store/pa10z4ngm0g83kx9mssrqzz30s84vq7k-hello-2.12.1.tar.gz",
        "stdenv": "/nix/store/jci7gw90lh2vdjaxkb6pzf9xp4v08wzs-stdenv-linux",
        "strictDeps": "",
        "system": "x86_64-linux"
      },
      "inputs": {
        "drvs": {
          "12aqc6cgq53348nizvwiybqr1l3h4wa7-stdenv-linux.drv": {
            "dynamicOutputs": {},
            "outputs": [
              "out"
            ]
          },
          "7r4i2k0iihf294fij4fharxs9ph4xy7i-gnumake-4.4.1.drv": {
            "dynamicOutputs": {},
            "outputs": [
              "out"
            ]
          },
          "n45fsyfbqfc0n2199kd56m65fr3375jv-hello-2.12.1.tar.gz.drv": {
            "dynamicOutputs": {},
            "outputs": [
              "out"
            ]
          },
          "rdi2zjmbw7565gc4pa4fkzvzpgwpaaj3-bash-5.3p9.drv": {
            "dynamicOutputs": {},
            "outputs": [
              "out"
            ]
          },
          "y7wnv3kycy9gpxzwmny1sppi7il9i31a-gcc-wrapper-15.2.0.drv": {
            "dynamicOutputs": {},
            "outputs": [
              "out"
            ]
          }
        },
        "srcs": [
          "l622p70vy8k5sh7y5wizi5f2mic6ynpg-source-stdenv.sh",
          "shkw4qm9qcw5sc5n1k5jznc83ny02r39-default-builder.sh"
        ]
      },
      "name": "hello-2.12.1",
      "outputs": {
        "out": {
          "path": "rc0j253ssqp88v8xzy5710ndz09kxnwy-hello-2.12.1"
        }
      },
      "system": "x86_64-linux",
      "version": 4
    }
  },
  "version": 4
}
```
可以看到，函数式的 Nix 表达式被解析为一个纯 JSON 数据，所有 `import`，函数调用，参数传递都被求值完毕，剩下一份确定、无歧义的构建配方。

3. 用户环境。\
Nix 将用户空间和安装空间分离。安装空间即 `/nix/store`，存放所有组件的实际文件；用户空间即 `~/.nix-profile`（加到 PATH 中的目录），本身只是一组指向 store 的符号链接。因此安装、卸载、升级、回滚操作都只是在增删和切换这些符号链接，store 中的文件始终不变。其中升级和回滚通过切换 generation 链接实现，借助 POSIX rename() 系统调用保证原子性——要么是旧环境，要么是新环境，不存在中间态。

4. 构建共享。\
原始论文中的 `nix-push / nix-pull` 已演进为 二进制缓存（binary cache） 机制。Nix 默认从官方维护的公共缓存 `cache.nixos.org` 拉取预构建产物，其覆盖了整个 nixpkgs 仓库的数十万个包。用户执行安装时，Nix 先计算 derivation 的哈希，去缓存中查找是否有匹配的预构建产物：有就直接下载，没有才在本地从源码编译。整个过程对用户完全透明，无需关心自己用的是"源码安装"还是"二进制安装"。


> Nix 源于荷兰语 `niks`，意思是 nothing，表示构建操作不会看到任何没有明确指定作为输入的内容。这句话该怎么理解呢？其实指的是不会访问没有指定的环境。

未完待续...
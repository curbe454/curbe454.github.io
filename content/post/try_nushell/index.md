---
title: "Try nushell and not feel good"
description: 
date: 2025-09-27T04:01:42+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
categories:
    - terminal
tags:
    - shell
    - nushell
    - awful
    - oh-my-posh
    - powershell
---
<!-- 9.26 - 9.27 -->
# nushell 踩坑记

## 9.26

听说 nushell 很好，我今天试了试，结果差点被劝退了。

看了一眼官方文档，感觉不错，居然还支持函数式，就一边看文档一边想办法转移我在其他 shell 的配置。

我首先就是使用我的 oh-my-posh. 这时候就踩坑了。我编写了一个这样的函数：

```nu
def set-theme [name] {
  let theme_dir = $"($env.SCOOP)/apps/oh-my-posh/current/themes"
  let theme_omp = $"($theme_dir)/($name).omp.json"
  let theme_nu = $"($theme_dir)/theme.nu"

  if not ($theme_nu | path exists) { touch $theme_nu }

  if ($theme_omp | path exists) {
    oh-my-posh init nu --config $"($theme_dir)/($name).omp.json" --print o> $theme_nu
  }
  source $theme_nu
}
```

思路很简单，就是利用传进来的字符串决定 theme 样式。

要是有懂 `nushell` 的就能一眼看出来，这里最重要的代码 `source $theme_nu` 是完全行不通的，因为这个代码是执行在函数的作用域里面而不是全局作用域。

我找了 Deepseek 求证了一下，它还给我出一些"在 `def` 之后加上 `--env`"，或者把"`source` 改成 `overlay use`"这样的馊主意。这个 `def --env` 是在当期望函数可以修改环境变量 `$env.*` 的时候使用，而 `overlay use` 是模块化包装，这样也不会把这个作用域给突破。

我想着这一点也不 shell-like，缺少这个功能，那就根本不是一个正宗的 shell. 我想着能否通过 alias 绕过，结果发现这样会形成 `alias theme = set-theme $name; source $theme_file` 这样的情况，这就要求 alias 也能输入参数。但是这是不符合 alias 语法的，除非我再引入一个新函数，但是这样还是不能绕过作用域。

我放弃了，于是把 source 语句放在了 `config nu` 的文件里，死马当活马医，要是不行我直接删了。结果就成功了，这个 config 是使用全局作用域的，好吧，我承认你是个 shell...

以上我折腾了俩小时的成果。。。途中我真的被这个"不能 parse"的错误提示给整急眼了，开始厌恶这个要命的静态解析，我写过 bash 和 powershell, 我哪能遭这种罪，甚至觉得 python 和 C 语言的时候多自由啊，这种这不让做那不让做的能叫 shell 嘛。。。只能说折腾起来真是坑啊。

## 9.27

不行了，我不玩了，我这就删除这个毒瘤。

我有这样一个奇异的 `powershell` 脚本：

```powershell
# OCaml
function opamact
{ (& opam env) -split '\r?\n' | ForEach-Object { Invoke-Expression $_ } }

function ocaml_init_wrapper {
  param(
    [string]$CommandName,
    [string]$ExecutableName = $null
  )

  if (-not $ExecutableName) {
    $ExecutableName = $CommandName
  }

  $functionDef = @"
  function global:$CommandName {
    if (-not `$env:OPAM_SWITCH_PREFIX)
    { opamact }

    & "$ExecutableName" @args
  }
"@

  Invoke-Expression $functionDef
}

ocaml_init_wrapper "ocaml" "ocaml.exe"
ocaml_init_wrapper "ocamlc" "ocamlc.exe"
ocaml_init_wrapper "ocamldoc" "ocamldoc.exe"
ocaml_init_wrapper "ocamlrun" "ocamlrun.exe"
ocaml_init_wrapper "ocamlopt" "ocamlopt.exe"
ocaml_init_wrapper "dune" "dune.exe"
ocaml_init_wrapper "utop" "utop.exe"
```

之所以要这么整，是因为假设直接 `opam env` 命令消耗的时间实在是有些长。本来我的电脑就不行，打开一个 `powershell` 对话就要大概 700 ms. 加上这个直接高达 1.2s, 这谁受得了；于是乎做了个按需加载。不止这个，我还有 conda 和 MSVC 环境呢，这些又应该怎么解决呢（当然这几个肯定要简单一些）。

我尝试了两个小时企图复现这个逻辑，结果发现这个东西的限制实在是太让人难受，alias 根本绑定不了 `source` 命令，这对我的动态加载是致命的。我最后最接近的版本是这个：

```nu
# OCaml
const opam_env_file = $"($code)/.opam_env.nu"
def get-opamenv [] {
  if not ($opam_env_file | path exists) { touch $opam_env_file }
  opam env --shell=powershell | lines | each {|e| str replace "$env:" "$env."} o> $opam_env_file
}
alias opamact = get-opamenv;
source $opam_env_file
```

是的，这样也未尝不可。每次使用了 `opamact` 或者 `get-opamenv` 之后记得再使用 `source` 命令就好。然后才能执行 `ocaml.exe`, `ocamlc.exe` 这些程序。。。一个命令只做一件事，十分符合 `nushell` 尊崇的 Unix 教义，对吧？
F\*\*k, can you understand why I just do these in `powershell`? 这个 `nushell` 的这一部分设计的一点也不好，让 shell 失去了它应该做到的动态加载的部分，给它加上了一把名叫 `parse` 的枷锁。我难受极了，感觉像被喂了 s\*\*t.

`scoop uninstall nu`, 启动！

---
title: '科学上网的基本原理'
description: '介绍代理内核、系统代理的生效方式'
pubDate: 'Aug 21 2026'
heroImage: '../../assets/MAIN-001/placeholder.png'
---

最高指示：互联网一定要能互联。

为了能访问真正的互联网，以Clash为首的代理客户端是我们常用的神奇妙妙工具。

但是在打开Clash后，并不是所有的应用都在按我们预期的运行，尤其是命令行环境或自己编写的脚本中。

本文在MacOS环境中，使用Clash Verge和mihomo内核，在不使用TUN模式的情况下，以实验为导向，解析常见的网络配置问题，包括：

- 系统代理修改了什么？
- 系统代理为何有时不生效？
- 通常在命令行中运行的程序为什么不会自动使用代理？
- 如何配置某软件的代理生效（以Git，HomeBrew，CodeX，Hugging face，Python.requests，Python.connect为例）？

# 代理客户端和内核
在一切开始前，我们先引入一个概念，代理客户端和内核。

代理客户端，即常见的ClashX，Clash Verge等，通常下载时自动包含了内核
代理内核，即mihomo，clash等

对此我们不多加赘述，但是我们必须认识到一点：
代理内核即可完成代理的全部工作，客户端只是让这一过程操作更加方便

# 基础原理
我们可以认为，在不使用TUN模式的情况下，应用程序都是使用代理的主动方，即应用程序要主动去寻找并配置代理，代理才会生效。

这一原理解释了为什么即使系统代理打开，还有应用程序不会通过代理。

在计算机网络模型中，代理软件工作在“应用层”，与其他应用平级。这也解释了为什么应用不会被动使用代理，以及Ping等更底层的命令为何不受代理影响。

应用程序主动寻找代理有三种常见情况：
- 显示参数指定
- 环境变量读取
- 读取系统配置

其中，
环境变量读取，如果未经说明或提到约定的、默认的等说法，指`HTTP_PROXY`、`HTTPS_PROXY`、`ALL_PROXY`。
读取系统配置的应用，会在“系统代理”打开时自动使用代理。

我们对此详细说明。

## Curl实验
在这个实验中，我们以Curl为例，探索以上三种情况。
在此过程中，我们全程打开Clash。
### Curl实验1：系统代理是否对Curl生效
首先我们确保Clash的系统代理关闭

我们直接在命令行执行命令
```
curl -v https://google.com
```
我们将获得类似如下内容的结果
```
* Host google.com:443 was resolved.
* IPv6: (none)
* IPv4: 142.251.34.206
*   Trying 142.251.34.206:443...
* Connected to google.com (142.251.34.206) port 443
```
显然，代理没有生效。

我们打开系统代理，重复上面的命令，会得到相同的结果。
因此我们认为：系统代理不会对Curl生效

### Curl实验2：显式指定参数是否对Curl生效
我们已经确认系统代理默认不生效，我们关闭系统代理。

Curl提供了参数来指定代理，我们运行以下命令：
```
curl -v -x http://127.0.0.1:7897 https://google.com
```
其中，7897为Clash Verge的默认端口，如果你使用mihomo，应该为7890，或者其他客户端自定义的或你手动指定的端口

我们将获得以下结果：
```
*   Trying 127.0.0.1:7897...
* Connected to 127.0.0.1 (127.0.0.1) port 7897
* CONNECT tunnel: HTTP/1.1 negotiated
* allocate connect buffer
* Establish HTTP proxy tunnel to google.com:443
> CONNECT google.com:443 HTTP/1.1
```
可以看到，代理生效了，并且返回了更多的详细内容，这是因为我们成功访问了目标网站。

由此我们可以得知，如果应用程序提供了参数供代理访问，我们可以使用。
修改持久化的配置文件的可能能让你不需要每次都指定，但原理类似，我们不做详细演示。
### Curl实验3：通用的环境变量是否对Curl生效
我们已经确认系统代理默认不生效，我们关闭系统代理。

我们在命令行运行以下命令：
```
export HTTP_PROXY=http://127.0.0.1:7897
export HTTPS_PROXY=http://127.0.0.1:7897
export ALL_PROXY=socks5h://127.0.0.1:7897
```
然后运行：
```
curl -v https://google.com
```
可以得到和实验2中相同的结果

由此我们得知，Curl会主动寻找并使用 约定 的环境变量，并使用代理。

## 对于环境变量的特别说明
`HTTP_PROXY`，`HTTPS_PROXY`，`ALL_PROXY`是常见的系统环境变量，可以被Curl，wget，python的requests库等程序自动读取，但这并不意味着它对所有程序有效。

我们仍然强调，使用代理时应用程序的主动行为，当你没有阅读程序的文档时，约定的常见环境变量不生效是完全正常的。

# 系统代理
通过上节的内容，我们了解到，系统代理不是总是生效，代理有多种指定方式，这必然会引起疑问：系统代理是怎么生效的？它的应用范围是什么？为什么浏览器总是可以使用系统代理？本节我们做详细解释。

请注意：本节与MacOS强相关，其他系统仅原理类似。

## 系统代理的原理
首先我们要明确，配置系统代理是代理客户端而非代理内核的行为。简而言之，“打开系统代理”的动作，是客户端运行了某种命令，将内核的端口和代理的协议告知操作系统。

在MacOS中，代理被通过`networksetup`应用被配置到`SystemConfiguration.framework`。

我们通过实验来说明这一过程，并尝试绕过Clash客户端来手动使用命令开启系统代理。

## networksetup实验1：系统代理修改的配置
我们关闭系统代理，运行命令：
```
scutil --proxy
```
该命令调取系统配置的API，获取代理相关的配置内容。

我们会得到类似如下的结果
```
<dictionary> {
  ExceptionsList : <array> {
    0 : 127.0.0.1
    1 : 192.168.0.0/16
    2 : 10.0.0.0/8
    3 : 172.16.0.0/12
    4 : 172.29.0.0/16
    5 : localhost
    6 : *.local
    7 : *.crashlytics.com
    8 : <local>
  }
  FTPPassive : 1
  HTTPEnable : 0
  HTTPSEnable : 0
  ProxyAutoConfigEnable : 0
  SOCKSEnable : 0
}
```
此时我们打开系统代理，运行同样的命令，我们会得到如下结果
```
<dictionary> {
  ExceptionsList : <array> {
    0 : 127.0.0.1
    1 : 192.168.0.0/16
    2 : 10.0.0.0/8
    3 : 172.16.0.0/12
    4 : 172.29.0.0/16
    5 : localhost
    6 : *.local
    7 : *.crashlytics.com
    8 : <local>
  }
  FTPPassive : 1
  HTTPEnable : 1
  HTTPPort : 7897
  HTTPProxy : 127.0.0.1
  HTTPSEnable : 1
  HTTPSPort : 7897
  HTTPSProxy : 127.0.0.1
  ProxyAutoConfigEnable : 0
  SOCKSEnable : 1
  SOCKSPort : 7897
  SOCKSProxy : 127.0.0.1
}
```
由此我们可知，“打开系统代理”的动作修改了`SystemConfiguration`的配置。

## networksetup实验2：手动模拟系统代理开启
我们手动关闭系统代理，重复实验1的前半部分，运行
```
scutil --proxy
```
可以看到与之前结果相同。

我们运行命令：
```
networksetup -setwebproxy Wi-Fi 127.0.0.1 7897
networksetup -setsecurewebproxy Wi-Fi 127.0.0.1 7897
networksetup -setsocksfirewallproxy Wi-Fi 127.0.0.1 7897
```
然后再次运行命令
```
scutil --proxy
```
我们会得到如下结果：
```
<dictionary> {
  ExceptionsList : <array> {
    0 : 127.0.0.1
    1 : 192.168.0.0/16
    2 : 10.0.0.0/8
    3 : 172.16.0.0/12
    4 : 172.29.0.0/16
    5 : localhost
    6 : *.local
    7 : *.crashlytics.com
    8 : <local>
  }
  FTPPassive : 1
  HTTPEnable : 1
  HTTPPort : 7897
  HTTPProxy : 127.0.0.1
  HTTPSEnable : 1
  HTTPSPort : 7897
  HTTPSProxy : 127.0.0.1
  ProxyAutoConfigEnable : 0
  SOCKSEnable : 1
  SOCKSPort : 7897
  SOCKSProxy : 127.0.0.1
}
```
我们发现，通过运行指定命令，我们将系统配置手动切换到了与“打开系统代理”相同的状态。

此时，我们通过浏览器访问IP查询网站，如 https://ip.sb/ 等，我们可以发现我们被配置到了代理指定的地址。
## 源码分析
此部分的源码位于 https://github.com/clash-verge-rev/sysproxy-rs/blob/main/src/macos.rs
我们可以看到其调用了
```
/// Fixed path prevents `PATH` substitution in privileged callers.
const NETWORKSETUP: &str = "/usr/sbin/networksetup";
```
并添加了"-setwebproxystate", "Wi-Fi"等参数。

## 生效范围
经过上述论证，我们可以得知系统代理的原理。但我们仍要强调，系统代理是一种被注册到系统层级的配置，仍然需要应用程序自身主动去通过系统API（例如scutil等）去获取，然后主动应用系统代理的配置，而不是在不知情的情况下已经使用了代理。

值得注意的是，以浏览器（包含内嵌浏览器内核的桌面应用）为首的桌面应用通常一般自动去获取该系统配置，因此产生了“打开系统代理=开始使用VPN“的假象。而事实上，只要代理内核在运行，用户就可以以指定参数、环境变量等方式使用代理；而即使打开系统代理，有些软件仍然会直连。

# 应用
我们以常用应用举例，如何实现对指定应用的代理配置。

## Git
如果不使用代理，在使用 Git 克隆或推送时，经常因为网络不稳定而失败。因此，对Git的代理配置很重要。

由 [官方文档](https://github.com/git/git/blob/master/Documentation/config/http.adoc) 可知，Git 兼容 Curl 的配置方式，这或许是因为其底层使用 `libcurl`，因此，其支持与 Curl 相同的环境变量，即`HTTP_PROXY`、`HTTPS_PROXY`、`ALL_PROXY`，你可以在当前命令或会话中指定它们，以达到效果。

另外，Git 支持使用`-c`参数，来指定当前 Git 命令的代理，例如
```
git -c http.proxy=http://127.0.0.1:7897 <command>
```
当然，由于我们需要在不同的软件中使用 Git，例如HomeBrew的依赖，IDE的命令行或可视化插件，每次配置显得格外麻烦，另外，Git在不使用代理时几乎总是失效，因此我们建议以下全局配置方案：
```
git config --global http.proxy http://127.0.0.1:7897
git config --global https.proxy http://127.0.0.1:7897
```
配置完成后，你可以使用
```
git config --get http.proxy
git config --get https.proxy
```
来查看配置结果。

默认情况下，我们也能在`~/.gitconfig`中查看到配置结果。

值得注意的是，改配置完成后，如果未开启代理（不必打开系统代理），git的远程命令将一定失败。

## HomeBrew
HomeBrew的情况与Git类似，其主要依赖 Curl和Git作为更新和分发工具，因此 约定 的环境变量仍然生效。

## CodeX
CodeX官方对于CodeX的配置没有详细的文档要求，但在26+的版本中，其似乎可以使用系统代理。Openai的工作人员在Github的 [Issue](https://github.com/openai/codex/issues/10555) 中指出，CodeX Desktop依赖CodeX Cli，对于Cli的修改也将对Desktop生效。

在其他 [Issue](https://github.com/openai/codex/issues/6060) 提到了在 `~/.config/codex/config.toml` 和 `~/.codex/.env` 等位置编写 约定 的环境变量，或直接在命令行中使用export等，一些用户认为其有效，但未找到明确的官方答复。

## Hugging face
Hugging face官方推荐通过，`HTTP_PROXY`、`HTTPS_PROXY`配置代理。
其`proxies`参数似乎已被废弃。

> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/weixin_52063276/article/details/137041956)

react native pod 第三方包或者 git clone 的时候遇到

```
git config --global --unset http.proxy
 
git config --global --unset https.proxy
 
git config --global http.sslBackend "openssl"
 
git config --global http.sslVerify false
```

两种解决方案

##### 方法一 修改计算机网络配置

由于使用 [IPv6](https://so.csdn.net/so/search?q=IPv6&spm=1001.2101.3001.7020 "IPv6") 的原因，可能会导致这一问题的出现

系统在解析 hostname 时使用了 [ipv6](https://so.csdn.net/so/search?q=ipv6&spm=1001.2101.3001.7020)

可以配置[计算机](https://so.csdn.net/so/search?q=%E8%AE%A1%E7%AE%97%E6%9C%BA&spm=1001.2101.3001.7020)不使用 IPv6，故使用以下命令：

```
$ git config --global -e
 
进入 git 的配置文件编辑界面
 
在该文件中加入如下内容：
 
 
[http]
        proxy = socks5://127.0.0.1:7890
[https]
        proxy = socks5://127.0.0.1:7890
```

后期可以再换回来

```
$ networksetup -setv6automatic Wi-Fi
```

##### 方法二 删除代理设置 

*   检查是否开了梯子网络代理，如果有先关闭；
*   在命令行输入如下命令

```
git config --global --unset http.proxy
 
git config --global --unset https.proxy
 
git config --global http.sslBackend "openssl"
 
git config --global http.sslVerify false
```

*   以上命令都完成了之后重启命令行窗口

##### 方法三 git 配置 HTTPS 代理

前提需要有梯子

打开终端在命令行输入如下命令

```
$ git config --global -e
 
进入 git 的配置文件编辑界面
 
在该文件中加入如下内容：
 
 
[http]
        proxy = socks5://127.0.0.1:7890
[https]
        proxy = socks5://127.0.0.1:7890
```

端口号是 你科学上网的端口号

![](https://i-blog.csdnimg.cn/blog_migrate/907a528b3ef5a9993c23dce0163a0c38.png) 

> 方法二，三适用于 git clone 报错的场景，方法一适用于 pod 报错例如 boost。
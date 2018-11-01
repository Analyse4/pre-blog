# minikube从入门到放弃指南

![icon](https://raw.githubusercontent.com/kubernetes/minikube/master/logo/logo.png)

折腾了一周如何装minikube，记录一下

- 安装 CentOS

  - 安装系统

  - 连接网络
    - 以太网
    - WI-FI

- 安装 ss

  - 配置文件

- 安装 privoxy

  - 配置文件

- 设置 ss 和 privoxy 开机自启

- 开启VT-x or AMD-v virtualization

- 安装 virtualbox

- 安装 kubectl

- 安装 minikube

- 参考


## 安装CentOS

##### **Environment**

- hardware: thinkpad-T410
- version:  CentOS 7.5.1804

(tips:发现了一个用于制作系统盘的软件Etcher)

##### 安装系统

镜像从相应官网下载，制作好系统盘后，think-T410开机时按F12可以进入引导设备选择界面，选择从U盘启动，会进入到CentOS7安装引导界面

![install-1](https://i.stack.imgur.com/qyqly.png)

在这里遇到了问题，无论以何种规格安装(mininal, graph)都会在安装过程中界面卡死，只能重启重装，之后还会在安装界面卡死，周而复始，起初我以为是目前的centos版本对于thinkpad-T410的硬件来说太新了，所以换ubuntu装了下(这里不得不提一句ubuntu的安装界面真的很友好)，发现也出现一样的问题，后来在StackExchange上找到了答案

> This happens often on computers with old graphics hardware. By default the system tries to use a 1024x768 framebuffer mode to start the system with, but this doesn't work with some old PCs.
>
> In this case, you can select **Troubleshooting** from the menu, and then select **Install <distro> in basic graphics mode**.

按照上面的方式试了下

![install-2](https://i.stack.imgur.com/BIGmo.png)

果然还不行，看来这台机器的图形卡真的太老了:smile_cat:

> On some *really* ancient computers, even this won't work. In that case you'll need to do a text mode installation. Do this by selecting **Install <distro>** but instead of pressing `Enter`, press `Tab` and add ` nomodeset text` to the end of the boot command line.
>
> [StackExange]: https://unix.stackexchange.com/questions/353896/linux-install-goes-to-blank-screen

只能按照最后一种方式安装，值得注意的是，`nomode text`这种模式是以命令行的交互方式安装，相当于彻底砍掉了图形界面，即使图形卡再旧也不会对这种模式有任何影响，结果成功

##### 连接网络

###### 以太网

First, type `nmcli d` command in your terminal for quick list **ethernet card** installed on your machine:

![connect-1](https://lintut.com/wp-content/uploads/2014/08/CentOS_7-network-setup.png)

Type `nmtui` command in your terminal to open **Network manager**. After opening Network manager chose **“Edit connection**” and press Enter (Use `TAB` button for choosing options).

![connect-2](https://lintut.com/wp-content/uploads/2014/08/CentOS_7-Network-manager-screen.png)

Now choose you network interfaces and click “**Edit**”

![connect-3](https://lintut.com/wp-content/uploads/2014/08/Edit-your-network-interfaces.png)

Choose “**Automatic**” in **IPv4 CONFIGURATION** and check **Automatically connect** check box and press `OK` and quit from Network manager.

![connect-4](https://lintut.com/wp-content/uploads/2014/08/Set-ip-adress-using-DHCP.png)

Reset network services:

```ssh
service network restart
```

Now your server will get IP Address from DHCP .

![connect-5](https://lintut.com/wp-content/uploads/2014/08/CentOS-7-check-ip-address.png)

###### 无线网

CentOS 7 以后采用**NetworkManager **来管理网络连接，首先需要确保**NetworkManager **能够管理无线设备从而支持WI-FI，要保证这一点，需要安装**NetworkManager -wifi**，并重启**NetworkManager **

```sh
yum install NetworkManager-wifi
systemctl restart NetworkManager
```

然后进入`nmtui`中的**Activate a connection**界面选择合适的WI-FI

## 安装ss

略

## 安装privoxy

安好了ss后，但它是**socks5**代理，我门在shell里执行的命令，发起的网络请求现在还不支持**socks5**代理，只支持**http／https**代理。为此需要安装**privoxy**代理，它能把电脑上所有**http**请求转发给ss

下载

```shell
wget http://www.silvester.org.uk/privoxy/source/3.0.26%20%28stable%29/privoxy-3.0.26-stable-src.tar.gz
```

解压

```shell
tar -vxf privoxy-3.0.26-stable-src.tar.gz
```

安装前需要`useradd privoxy`创建一个用户privoxy

```shell
useradd privoxy
```

然后依次执行如下三条命令

```shell
autoheader && autoconf
./configure
make && make install
```

修改配置文件，该之前备份配置文件

```shell
cp  /etc/privoxy/config /etc/privoxy/config.bak
```

查看`vim /etc/privoxy/config`文件，

先搜索关键字:`listen-address`找到`listen-address 127.0.0.1:8118`这一句，保证这一句没有注释，*8118*就是将来http代理要输入的端口。

然后搜索`forward-socks5t`,将`forward-socks5t / 127.0.0.1:1080 .`此句的注释去掉

执行如下命令启动**privoxy**，参考[官网,不同的平台对应不同的方法](https://link.jianshu.com/?t=https%3A%2F%2Fwww.privoxy.org%2Fuser-manual%2Fstartup.html)(这里针对**CentOS**平台，有可能装在**/usr/local/sbin**下)

```shell
/usr/sbin/privoxy --user privoxy /etc/privoxy/config
```

执行`vim /etc/profile`,添加如下三句

```shell
export http_proxy=http://127.0.0.1:8118
export https_proxy=http://127.0.0.1:8118
export ftp_proxy=http://127.0.0.1:8118
```

如果不能访问，请重启机器，依次打开**shadowsocks**和**privoxy**再测试

## 设置 ss 和 privoxy 开机自启

##### Old method

由于**ss**和**privoxy**服务需要时刻保持运行，所以想到把这两个服务配置成开机自启的服务，首先找到的是通过配置**/etc/rc.local**的方法，由于

> /etc/rc.local or /etc/rc.d/rc.local are no longer executed by default due to systemd-changes.
> to still use those, you need to make /etc/rc.d/rc.local executable

具体做法如下

```shell
chmod +x /etc/rc.d/rc.local
```

 假设需要开机自启的脚本为**autobootup.sh**，那么应该在**/etc/rc.local** (which is a symlink to **/etc/rc.d/rc.local**)中添加以下代码

```shell
/your/path/autobootup.sh
```

##### Recommended method

但这样做其实有一些潜在问题，因为打开**/etc/rc.local**文件可以看到

>  #!/bin/bash 
>
> THIS FILE IS ADDED FOR COMPATIBILITY PURPOSES
>
> It is highly advisable to create own systemd services or udev rules
>
> to run scripts during boot instead of using this file.
>
> In contrast to previous versions due to parallel execution during boot
>
> this script will NOT be run after all other services.
>
> Please note that you must run 'chmod +x /etc/rc.d/rc.local' to ensure     
>
> that this script will be executed during boot.                                                                                                                                                                                                         

可以得知，这种方法是为了兼容性保留下来的，CentOS 7中建议使用创建自己的**systemd services**的方式解决这种需求

1.Let us first create a custom script to be run at system boot automatically.

> #!/bin/bash
>
> #some services for auto run during boot
>
> echo "This is a custom script to auto run during boot" > /var/tmp/script.out
> echo "The time the script run was -->  `date`" >> /var/tmp/script.out
>
> #run ss-cli
>
> nohup sslocal -c /etc/shadowsocks.json /dev/null 2>&1 &
>
> #run privoxy
>
> nohup /usr/local/sbin/privoxy --user privoxy /usr/local/etc/privoxy/config

2.Add execute permission(if it’s not already set).

```shell
chmod +x /your/path/yourscript.sh
```

Create a new service unit file at **/etc/systemd/system/sample.service** with below content. The name of the service unit is user defined and can be any name of your choice.

> [Unit]
> Description=auto run some scripts during boot
>
> [Service]
> Type=forking
> ExecStart=/root/myscripts/autobootup.sh
> TimeoutStartSec=0
>
> [Install]
> WantedBy=default.target

这里特别要注意的是**Type=forking**，由于这里开机自启动的脚本中本质是通过该脚本启动两个进程，所以**Type**对应的域应该填**forking**

Reload the systemd process to consider newly created sample.service OR every time when sample.service gets modified.

```shell
systemctl daemon-reload
```

Enable this service to start after reboot automatically.

```shell
systemctl enable sample.service
```

Start the service.

```shell
systemctl start sample.service
```

Reboot the host to verify whether the scripts are starting as expected during system boot.

```shell
systemctl reboot
```

## 开启VT-x or AMD-v virtualization

VT-x or AMD-v virtualization must be enabled in your computer’s BIOS.

## 安装virtualbox

去官网下载相应的rpm包(**Oracle Linux 7 / Red Hat Enterprise Linux 7 / CentOS 7**)

把相应的rpm包传到服务器上

执行

```shell
yum install VirtualBox-5.2-5.2.20_125813_el7-1.x86_64.rpm
```

这里特别说明下rpm命令与yum命令的区别

> Next step is to actually install the software package and you have the two options `yum` and `rpm`. The method `rpm -i skype-4.2.0.13-2.fc20.i686.rpm` will try to install the package but can complain about unmet dependencies. `yum install skype-4.2.0.13-2.fc20.i686.rpm` will check for dependencies and will try to resolve them and provide you a detailed installation plan which you can accept to actually install the package including all its dependencies.
>
> [StackExange]: https://unix.stackexchange.com/questions/343919/how-to-install-a-rpm-package-in-a-right-way

之后运行virtualbox的时候可能会出现某个模块未编译的问题，安装提示装一个类似于内核头文件的包后重新编译该模块即可，具体细节忘记截图了，这里就不展开了

## 安装 kubectl

**notice**: You must use a kubectl version that is within one minor version difference of your cluster. For example, a v1.2 client should work with v1.1, v1.2, and v1.3 master. Using the latest version of kubectl helps avoid unforeseen issues.

**CentOS, RHEL or Fedora**

```shell
$ cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
$ yum install -y kubectl
```

## 安装minikube

##### Environment

version: 0.30.0

##### Install

###### Normal

```shell
$ curl -Lo minikube https://storage.googleapis.com/minikube/releases/v0.30.0/minikube-linux-amd64 && chmod +x minikube && sudo cp minikube /usr/local/bin/ && rm minikube
```

可以看到，minikube整个安装过程非常简单，按照官方文档的**quick start**执行

```shell
$ minikube start
Starting local Kubernetes v1.7.5 cluster...
Starting VM...
SSH-ing files into VM...
Setting up certs...
Starting cluster components...
Connecting to cluster...
Setting up kubeconfig...
Kubectl is now configured to use the cluster.
```

这是真正噩梦的开始，直接按照以上命令执行，会卡在**Starting cluster components...**

为了方便理解整个过程，先来看一下minikube的架构

![minikube-1](https://i0.wp.com/blog.ichasco.com/wp-content/uploads/2018/06/minikube-architecture.jpg?resize=810%2C455&ssl=1)

在启动minikube的时候，首先会向gcr.io(待考证)拉取一个ISO镜像，因为之前配置好的缘故，这里是可以拉取成功的，之后会利用之前安装好的hypervisor在宿主机中启一个virtual machine，之后准备在该virtual machine中构建整个kubernetes，问题就发生在这一步，用于构建kubernetes的所有组件是以镜像形式存在的，所以在构建之初，会在virtual machine里向gcr.io拉取所有组件镜像，虽然之前通过配置，可以使得宿主机从gcr.io拉镜像下来，但在virtual machine中却没有做相关配置，这导致在virtual machine中是无法从gcr.io中拉取镜像的

明白了这一点，就想到把virtual machine中的docker daemon配置一个代理，从而使得virtual machine中的docker daemon通过代理从gcr.io拉取镜像，minikube也提供相关的命令选项来实现这一点

```shell
$ minikube start --docker-env HTTP_PROXY=http://yourhost:yourport --docker-env HTTPS_PROXY=http://yourhost:yourport --docker-env NO_PROXY=192.168.99.0/24
```

特别注意的是**NO_PROXY**，上述命令已经使得virtual machine中的docker daemon走代理，但同时要保证容器之间的通信不能走代理，因为每个容器都是内网IP，一旦走代理，代理服务器将永远找不到要通信的容器对象，minikube中所有组件容器在**192.168.99.0/24**这个网域，所以这个网域的所有IP不应走代理

但是执行上述命令后仍会发现卡在**Starting cluster components...**

这是因为安装在宿主机的**kubectl**是全局走代理的，在启动minikube后，**kubectl**需要和virtual machine中的kubernetes cluster通信，问题就出在这一步，这一步通信现阶段依然是走代理的，但由于virtual machine的ip是内网IP，所以通过代理去找这个内网IP是永远找不到的，导致**kubectl**无法建立通信，卡在这里，为了解决这个问题，让kubernetes访问该virtual machine时不走代理，直接通信，执行如下命令

```shell
#192.168.99.100是运行minikube cluster的virtual machine的IP
$ export no_proxy=192.168.99.100
```

在见到如下信息后

```shell
Starting local Kubernetes v1.7.5 cluster...
Starting VM...
SSH-ing files into VM...
Setting up certs...
Starting cluster components...
Connecting to cluster...
Setting up kubeconfig...
Kubectl is now configured to use the cluster.
```

执行如下命令(待考证)

```shell
$ export no_proxy=$no_proxy,$(minikube ip)
$ export NO_PROXY=$no_proxy,$(minikube ip)
```

之后按照如下命令测试

```shell
$ kubectl run hello-minikube --image=k8s.gcr.io/echoserver:1.4 --port=8080
deployment "hello-minikube" created
$ kubectl expose deployment hello-minikube --type=NodePort
service "hello-minikube" exposed

# We have now launched an echoserver pod but we have to wait until the pod is up before curling/accessing it
# via the exposed service.
# To check whether the pod is up and running we can use the following:
$ kubectl get pod
NAME                              READY     STATUS              RESTARTS   AGE
hello-minikube-3383150820-vctvh   1/1       ContainerCreating   0          3s
# We can see that the pod is still being created from the ContainerCreating status
$ kubectl get pod
NAME                              READY     STATUS    RESTARTS   AGE
hello-minikube-3383150820-vctvh   1/1       Running   0          13s
# We can see that the pod is now Running and we will now be able to curl it:
$ curl $(minikube service hello-minikube --url)
CLIENT VALUES:
client_address=192.168.99.1
command=GET
real path=/
...
$ kubectl delete service hello-minikube
service "hello-minikube" deleted
$ kubectl delete deployment hello-minikube
deployment "hello-minikube" deleted
$ minikube stop
Stopping local Kubernetes cluster...
Machine stopped.
```

可以发现是成功的

###### Unnormal

这里参考阿里易立的思路，提供一种不配置代理，直接启minikube的方案

之前之所以没办法按正常的方式启是因为minikube总需要从gcr.io拉镜像，导致镜像一直拉不下来，进而失败，易立的思路很简单，就是把minikube源码中涉及访问gcr.io的部分替换为访问阿里云的镜像源，从根本上解决了问题(需要注意的是，如果之后继续按照github上minikube项目中**quick start**部分去测试的话，是无法成功的，因为命令`kubectl run hello-minikube --image=k8s.gcr.io/echoserver:1.4 --port=8080`本身就是从gcr.io拉镜像，而此时virtual machine中的docker daemon是没有配置代理的)

解决办法暂时能想到的是从国内镜像源拉景象，这就涉及到可能需要重新docker login一下(具体有待尝试)

[github](https://github.com/AliyunContainerService/minikube/tree/aliyun-v0.30.0) - 阿里云魔改版minikube分支

[云栖社区](https://yq.aliyun.com/articles/221687?spm=a2c4e.11153940.blogcont221687.227.41a77733GGXJ0Y&p=1#comments) - 易立云栖社区BLOG

## 参考

[简书](https://www.jianshu.com/p/41378f4e14bc) - joncc的简书BLOG

[就不怕忘记了](https://fatfatson.github.io/2018/07/23/mac%E4%B8%8A%E5%AE%89%E8%A3%85mimikube/) - mac上安装minikube及排错记

[github](https://github.com/kubernetes/minikube/issues/530) - Starting minikube behind a proxy

[github](https://github.com/Unknwon/wuwen.org/issues/20) - 代理环境下在 macOS 上安装 Minikube 小记

[github](https://github.com/kubernetes/minikube/blob/master/docs/http_proxy.md) - Using Minikube with an HTTP Proxy

[What is the difference between 0.0.0.0, 127.0.0.1 and localhost?](https://stackoverflow.com/questions/20778771/what-is-the-difference-between-0-0-0-0-127-0-0-1-and-localhost)




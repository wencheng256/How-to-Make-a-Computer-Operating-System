## 第二章: 配置开发环境

第一步，配置一个可运行的开发环境。 使用Vagrant和Virtualbox，我们将可以在任何操作系统 (Linux, Windows or Mac)上测试我们自己的OS

### 安装 Vagrant

> Vagrant 是一款免费的开源软件，可以用来创建和配置虚拟开发环境。 你可以把他看做VirtualBox的一层封装。
> 
Vagrant 可以帮我们在任意平台搭建一个干净的虚拟环境，第一步是下载安装适合你操作系统的Vagrant http://www.vagrantup.com/.

### 安装Virtualbox

> Oracle 虚拟机 VirtualBox 是一个为x86和64位机开发的可视化软件包。

Vagrant 需要基于VirtualBox工作，你可以在https://www.virtualbox.org/wiki/Downloads下载并安装。

### 启动并测试你的开发环境

当Vagrant 和Virtualbox 安装完毕后，你需要现在Vagrant 的 ubuntu lucid32镜像：

```
vagrant box add lucid32 http://files.vagrantup.com/lucid32.box
```

lucid32镜像准备好以后，我们需要使用一个*Vagrantfile* 定义我们的开发环境, [创建一个名为 *Vagrantfile*的文件](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/Vagrantfile). 这个文件定义了我们所需要的环境：nasm, make, build-essential, grub 和qemu.


开启你的虚拟机:

```
vagrant up
```

你现在可以使用ssh连接到virtual box：

```
vagrant ssh
```


连接到Vagrantfile的文件夹将被你的游客虚拟机默认挂载到*/vagrant*路径下 (在Ubuntu Lucid32环境下):

```
cd /vagrant
```

#### 构建并测试我们的操作系统

 [**Makefile**](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/Makefile) 文件定义了一些用来构建内核、用户库和一些终端程序的基础配置

构建:

```
make all
```

使用qemu测试我们的OS:

```
make run
```

可以在 [QEMU Emulator Documentation](http://wiki.qemu.org/download/qemu-doc.html)查看qemu的文档 

你可以使用Ctrl-a命令退出模拟器

# [译:]如何在Linux上检查可用磁盘空间


本文翻译自itsfoss的文章，主要介绍了怎样在linux下查看硬盘的可用空间。

<!--more-->

{{< admonition type=tip title="注意" open=true >}}
本文翻译自 [It's FOSS](https://itsfoss.com/)的文章，他的文章非常原汁原味，既通熟易懂又利于学习英语.
原文链接：<https://itsfoss.com/check-free-disk-space-linux/>.
{{< /admonition >}}

**我已经使用了多少磁盘空间？**

在Linux下最简单的检查可用磁盘空间方法是使用`df`命令。df命令一目了然地向你展示Linux下可用的磁盘空间信息。
```shell
df -h
```
添加`-h`选项，它会以人性化(human-readable)的方式向你展示磁盘空间内容(MB或GB)。
下面是我仅安装了Linux和加密磁盘的Dell XPS系统下df命令的输出内容：

![Linux下通过df命令检查磁盘可用空间](https://itsfoss.com/content/images/wordpress/2020/11/free-disk-space-linux-df-command-output.png)

如果上面的命令你不太理解，没关系，我将向你推荐一些关于在Linux下检查磁盘空间的方法。**我将介绍一些图形化操作给Linux桌面使用者**。
### 方法一：使用df命令检查磁盘可用空间（并明白其输出内容）
当你使用df命令检查磁盘空间后，它会显示一堆关于“文件系统”的大小、已用空间和可用空间的内容。你的磁盘通常为以下之一：
- /dev/sda
- /dev/sdb
- /dev/nvme0n1p

上面并不是一成不变的规则，但它为您提供了一种参考，让你可以从一大堆显示中识别出实际的磁盘。
你的Linux系统也许在你的磁盘上有许多分区，如boot、EFI、root、swap和home等。在这种情况下，这些分区会在“磁盘名称”的末尾显示一个数字，例如/dev/sda1，/dev/nvme0n1p2等。
您可以从挂载点确定哪个分区被用于什么目的。 如根目录(Root)挂载在/下，EFI挂载在/boot/EFI下等。
在我这里，我已经用完root下232 GB磁盘空间的41％，如果你有2-3个大的分区(像root、home等)，您将需要在此处进行计算：

![明白df命令的输出](https://itsfoss.com/content/images/wordpress/2020/11/df-command-output.png)

- **tmpfs**: `tmpfs`(临时文件系统) 被用来保持文件到虚拟内存中，你可以忽略这个虚拟文件系统。
- **udev**: `udev 文件系统`被用来存储一些插入到你的系统上的设备信息(如USB、网卡、光驱等)，你也可以忽略它。
- **/dev/loop**: 这些都是循环设备。由于一些快照应用你在检查磁盘空间时将会看到许多关于这样的内容。Loops是虚拟设备，可以将普通文件作为块设备进行访问。由于有循环设备，快照应用程序将被沙盒式地存储在其自己的虚拟磁盘中。 因为它们位于根目录下，因此您无需分别计算其已用磁盘空间。
#### 磁盘空间不足？ 检查你是否已安装所有磁盘和分区
请记住，df命令仅显示已挂载文件系统的磁盘空间。如果你在同一个磁盘下使用了不止一个Linux发行版系统(或者操作系统)，或者系统上有多个磁盘。你则需要先挂载它们，然后再查看这些分区和磁盘上的可用空间。
举个例子，我的 [Intel NUC](https://itsfoss.com/install-linux-on-intel-nuc/)上的两个固态硬盘安装了4到5个Linux发行版系统，只有当我显式地挂载它们时，它们的磁盘信息才会显示。

![](https://itsfoss.com/content/images/wordpress/2020/11/df-command-ubuntu-1.png)
![](https://itsfoss.com/content/images/wordpress/2020/11/df-command-ubuntu.png)

你可以使用 `lsblk` 命令来查看所有磁盘信息和分区信息。

![](https://itsfoss.com/content/images/wordpress/2020/11/lsblk-command-to-see-disks-linux.png)

有了磁盘分区名称后，你就可以用以下方式挂载它：
```shell
sudo mount /dev/sdb2 /mnt
```
还有一个关于在Linux上检查硬盘空间的好主意，让我们看看如何以图形化形式进行操作。
*推荐阅读：[使用 `duf` 命令行工具检查你的磁盘使用空间(可替换 du 和 df 命令)](https://itsfoss.com/duf-disk-usage/)*
### 方法二：图形化操作检查磁盘可用空间
在Ubuntu中，使用Disk Usage Analyzer工具以图形方式检查可用磁盘空间要容易得多。

![Disk Usage Analyzer Tool](https://itsfoss.com/content/images/wordpress/2020/11/disk-usage-analyzer-tool-linux.jpg)

您将会看到所有实际的磁盘和分区。 您可以通过单击某些分区来挂载它们。 Disk Usage Analyzer Tool 会显示所有已挂载分区的磁盘使用情况。

![Disk usage check](https://itsfoss.com/content/images/wordpress/2020/11/free-disk-space-ubuntu-desktop-800x648.png)

#### 使用GNOME磁盘实用工具检查可用磁盘空间

![GNOME Disks Tool](https://itsfoss.com/content/images/wordpress/2020/11/disks-tool-linux.jpg)

打开工具然后选择一个磁盘，选择一个分区来查看磁盘剩余空间，如果一个分区没有被挂载，通过点击“play”按钮来挂载它。

![剩余磁盘空间在Ubuntu桌面显示](https://itsfoss.com/content/images/wordpress/2020/11/free-disk-space-check-ubuntu-desktop.png)

我认为所有主流的Linux发行版环境都有相关的图形化操作工具来检查磁盘空间，你可以在Linux桌面上通过搜索菜单来找到它们。
### 总结
有许多方法和工具来检查磁盘空间，我只是向您展示了最常用的命令行和GUI方法。
我还解释了一些可能会困扰您理解磁盘使用情况的事情。 希望你会喜欢。
如果您有任何疑问或建议，可以在评论部分告诉我。

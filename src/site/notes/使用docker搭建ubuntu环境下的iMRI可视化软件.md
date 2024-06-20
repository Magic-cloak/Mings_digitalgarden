---
{"dg-publish":true,"permalink":"/使用docker搭建ubuntu环境下的iMRI可视化软件/","noteIcon":"","created":"2024-05-04T17:53:29.531+08:00","updated":"2024-05-10T17:38:46.175+08:00"}
---


# 搭建过程记录
## 开始的设想
最开始的计划是使用docker装配仅能在ubuntu环境下使用的fsl软件的容器。
但是这会导致不同操作系统下可能无法使用在win环境下编写的iMRI可视化程序。
因此师兄提出，可不可以将先前在win下运行的可视化程序移植到linux下，和fsl一起打包成容器，完成对应操作时通过点击设计好的ui按钮，通过fsl完成对应功能（例如：MRI图像除去头皮颅骨、大脑分割为白质、灰质、脑脊液区），这样无论在linux、win、mac环境下都可以通过docker容器使用该软件。

---

## 第一次尝试（成功一部分）
第一次尝试是参考了csdn上的博客，安装好docker后，通过cmd命令
```
docker pull neurodata/fsl_1604

docker run -itd --name fsl_1604  -v /d/docker/fsl_1604:/shared_data neurodata/fsl_1604 bash
```
直接利用他人已经配置好的装载fsl软件的镜像配置容器。
完成此操作后，不需要再配置PATH，可进入容器，使用命令
```
docker exec -it fsl_1604 /bin/bash # win中使用进入容器

bet /shared_data/test.nii.gz /shared_data/test_out.nii.gz # 脱去头皮

fast -s 1 -t 1 -n 3 -H 0.1 -I 4 -l 20.0 -o /shared_data/T1_image_test /shared_data/test_out.nii.gz # 分割脑区
```
完成操作，输出操作后的文件。能够完成操作，但是没有整合在同一程序、环境中。这时候得知师兄希望将程序全部移植到docker容器中。

---
## 第二次尝试（失败）
第二次尝试打算利用第一次尝试已经安装好fsl的容器。
在将win环境下的可视化程序所需的python函数包导出为requirement.txt，再使用容器pip函数包时发现：由于该镜像创建时间较早，系统为ubuntu16.04，环境较旧，pip出现了各种各样的bug。经过换源、使用anaconda配置、装新版open-ssl，仍未解决问题。多次尝试无果后，打算自己重新使用一个纯净的ubuntu22.04镜像，全部自行搭建。

---

## 第三次尝试（成功的基础）
使用docker指令
```
docker pull ubuntu:22.04
```
拉取ubuntu2204纯净版镜像（仅不到80M）
```
docker run -it --privileged=true -p 3000:22 --name gui_iMRI -v /e/Tools/Docker/gui_MRI:/shared_data ubuntu:22.04 bash
```
利用镜像创建好容器后，经过换源、安装指定版本python（3.11.8）后，使用命令
```
pip install -r /shared_data/requirements.txt
```
顺利将所需的python函数包导入，运行程序
```
python3 MRI_SN_beta_v1_main.py
```
结果pyqt5报错，搜索资料发现是linux下libqxcb.so与win下不同，需要重新apt-get。通过命令
```
sudo apt-get install libqxcb*
```
将文件全部安装后，该问题解决。但是出现新的问题。ubuntu系统显示没有$DISPLAY无法初始化编写程序的ui。
查阅资料后得知，docker容器无法直接获得容器中的gui，只能通过命令行操作。
然后我开始查找有没有办法将docker中的容器可视化的方法

---

## 第四次尝试（失败）
查找资料后发现：kasm似乎正好能解决这个问题
https://github.com/linuxserver/docker-kasm
kasm可以直接在容器中搭建好ubuntu可视化桌面，自然可以将程序ui显示出来。
按照kasm官方文档搭建后，在最后一步，我在浏览器中却始终无法将桌面初始化出来。
经过一番搜索，我在kasm的github一则issue中发现：
https://github.com/linuxserver/docker-kasm/issues/47
kasm官方人员称：windows并不是kasm测试的平台。
这个方法到这里中断。

---

## 第五次尝试（成功）
经过多次查找资料，我确信没有能够不借助本机软件，仅通过docker实现ubuntu系统下图像化界面显示的方法。
我发现mobaxterm是一个折中的办法：
https://mobaxterm.mobatek.net/download-home-edition.html
win系统下可以使用官方免安装程序显示docker中ubuntu需要显示的ui窗口。原理是ip远程控制，虽然宿主机和目标机为同一设备（笑）

有了初步思路，我在第三次尝试的基础上继续。
```
docker run -it --privileged=true -p 3000:22 --name gui_iMRI -v /e/Tools/Docker/gui_MRI:/shared_data ubuntu:22.04 bash
```
通过安装ssh service，设置端口并允许密码登录。
```
apt-get install openssh-server
echo "Port 22">>/etc/ssh/sshd_config
echo "PermitRootLogin yes">>/etc/ssh/sshd_config
service ssh start
service ssh status
```
启动ssh服务，并检查端口无误后。在ubuntu中输入
```
passwd
```
随后输入密码，设置容器root用户密码（没有密码外部无法登陆）

打开mobaxterm，新建一个连接。宿主机IP即为本机IP，宿主机的3000端口即为容器的22端口。故SSH连接参数按照以上参数输入即可。**注意修改端口号**。
连接建好之后，输入用户名：root，加上上面设置的密码便可远程登录到容器里了。到此SSH连接成功。
使用ubuntu中的图形化文本编辑器gedit，发现顺利出现gedit的ui界面。运行可视化程序也成功出现ui，并且能够完成指定任务。图像化界面设置成功！

---
## 手动安装fsl（巧妙地成功）
由于本身是自行搭建docker容器，需要自行安装fsl软件，参照fsl官方wiki：
https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/FslInstallation
安装ubuntu系统下的fsl，完成注册后，使用fslinstaller.py，进行安装。安装完成后，按照fsl官方wiki的方法设置环境变量和路径。
然而在命令行输入fsl，启动软件时，报错没有$env(USER)环境变量。这个bug我通过多种方式添加环境变量USER后，都没有解决。
突发奇想尝试修改调用该环境变量的函数fslstart.tcl，将原本的
```
set USER       $env(USER)
```
修改为
```
set USER       "root"   
```
后，再次在命令行中输入fsl，结果fsl软件顺利启动。
## ubuntu c++编程环境搭建

```bash
sudo apt install net-tools
sudo apt-get install openssh-server

sudo passwd root  #修改密码

sudo vi /etc/apt/sources.list
sudo apt-get update
sudo apt-get -f install #修复受损软件包
```

### 修改国内源`sources.list`

```bash
vi /etc/apt/sources.list
#修改后更新源
sudo apt-get update
```

```bash
###清华源
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-updates main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-backports main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-backports main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-security main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-security main restricted universe multiverse
deb http://security.ubuntu.com/ubuntu xenial-security main
# deb-src http://security.ubuntu.com/ubuntu xenial-security main



deb http://mirrors.aliyun.com/ubuntu/ focal main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ focal-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-security main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ focal-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-updates main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ focal-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-proposed main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ focal-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-backports main restricted universe multiverse
```

### 编译环境

```bash
sudo apt install cmake git cmake-gui build-essential 
```

### opencv

```bash
sudo apt install libgtk2.0-dev libavcodec-dev libavformat-dev libjpeg-dev libswscale-dev libtiff5-dev pkg-config
```

### boost

```bash
sudo apt-get install libboost-all-dev 

#mingW64 Qt
cd /D d:\boost_1_73_0
bootstrap gcc
b2 -j8 install --prefix=c:\boost_1_73_0\mingW64  --build-dir=.\tmp --build-type=complete threading=multi link=shared address-model=64 toolset=gcc stage
```

### 安装Visual Studio Code

```bash
sudo add-apt-repository ppa:ubuntu-desktop/ubuntu-make
sudo apt-get update
sudo apt-get install ubuntu-make
umake ide visual-studio-code
```

### Qt5

```bash
sudo apt-get install  qt5-default qtcreator
```

### opencv for MingW64

```powershell
cd opencv-4.3.0
mkdir release && cd release

cmake.exe -G "MinGW Makefiles" -D CMAKE_BUILD_TYPE=Release  -D CMAKE_INSTALL_PREFIX= C:\\opencv4.3\\mingw81_64  -D CMAKE_PREFIX_PATH= D:\\Qt\\5.15.0\\mingw81_64\\lib\\cmake\\Qt5  -D WITH_TBB=ON  -D WITH_V4L=ON  -D WITH_QT=ON  -D WITH_GTK=ON  -D WITH_OPENGL=ON  -D WITH_VTK=ON  -D OPENCV_GENERATE_PKGCONFIG=YES ..


cmake.exe -G "MinGW Makefiles" -DOPENCV_EXTRA_MODULES_PATH= D:\\opencv_contrib-4.3.0\\modules D:\\opencv-4.3.0 -D CMAKE_BUILD_TYPE=Release  -D CMAKE_INSTALL_PREFIX= C:\\opencv4.3\\mingw81_64  -D CMAKE_PREFIX_PATH= D:\\Qt\\5.15.0\\mingw81_64\\lib\\cmake\\Qt5  -D WITH_TBB=ON  -D WITH_V4L=ON  -D WITH_QT=ON  -D WITH_GTK=ON  -D WITH_OPENGL=ON  -D WITH_VTK=ON  -D OPENCV_GENERATE_PKGCONFIG=YES ..

mingw32-make -j 8


cmake.exe -G "MinGW Makefiles" -DCMAKE_BUILD_TYPE = Debug ..
```

### opencv for ubuntu

```bash
cd opencv-4.3.0
mkdir release && cd release

sudo cmake -DOPENCV_EXTRA_MODULES_PATH=~/opencv_contrib-4.3.0/modules ~/opencv-4.3.0 -D CMAKE_BUILD_TYPE=Release \
-D CMAKE_INSTALL_PREFIX=/usr/local \
-D CMAKE_PREFIX_PATH=/usr/lib/x86_64-linux-gnu/cmake/Qt5 \
-D WITH_TBB=ON \
-D WITH_V4L=ON \
-D WITH_QT=ON \
-D WITH_GTK=ON \
-D WITH_OPENGL=ON \
-D WITH_VTK=ON \
-D OPENCV_GENERATE_PKGCONFIG=YES ..

make -j 8
```

### opencv  for Panel

```bash
source /opt/pancake-core-sdk/environment-setup-armv7ahf-neon-poky-linux-gnueabi 
cd ~/opencv-4.3.0/platforms/linux
mkdir -p build_arm
cd build_arm

cmake -DOPENCV_EXTRA_MODULES_PATH=~/opencv_contrib-4.3.0/modules ~/opencv-4.3.0 -D CMAKE_BUILD_TYPE=Release -D CMAKE_INSTALL_PREFIX=$SDKTARGETSYSROOT -DCMAKE_TOOLCHAIN_FILE=../arm-gnueabi.toolchain.cmake ../../..


make -j 8



```

### googleTest

```bash
sudo apt-get install libssl-dev uuid-dev libmosquitto-dev libgtest-dev libgmock-dev
sudo apt-get install mosquitto
sudo apt-get install lcov
```

### Vscode Remote-ssh

```bash
cd ~
ssh-keygen
cd .ssh
cat id_rsa.pub >> authorized_keys
cat id_rsa #拷贝至 windows 下.ssh内.\id_rsa-remote-ssh
```

### Remote - SSH配置文件(私钥)：

```bash
Host 192.168.56.101
  HostName 192.168.56.101
  User root
  IdentityFile  .\id_rsa-remote-ssh
```

### Git Graph

### 配置git server（公钥）

### 添加用户

```bash
sudo adduser test
```

### 共享盘 `/home/disk_500G`

```bash
sudo chown 用户名:用户名 -R /home/disk_500G/XXX
sudo chmod -R o=- /home/disk_500G/XXX # 仅自己可访问XXX目录
```

### bitbucket配置ssh

```bash
cd ~
ssh-keygen -t rsa -C "yzhu798@XX.com" -f id_rsa_bitbucket
mv id_rsa_bitbucket* .ssh
cd .ssh
cat id_rsa_bitbucket.pub #拷贝并添加至bitbucket
rm id_rsa_bitbucket.pub  #删除公钥
sudo chmod -R o=- ~ # 仅自己可访问自己目录
```

### 配置windows的ssh登陆

```bash
cd ~
ssh-keygen  -f id_rsa_remote_ssh
mv id_rsa_remote_ssh* .ssh
cd .ssh
cat id_rsa_remote_ssh.pub >> authorized_keys
cat id_rsa_remote_ssh #拷贝至 windows 下.ssh内.\id_rsa_remote_ssh
```

### Vscode安装remote-ssh配置

**config**

```bash
Host 1.1.1.1
  HostName 1.1.1.1
  User hu ##修改为你的用户名
  IdentityFile  .\id_rsa_remote_ssh
```

### 忘记ubuntu密码

```
1. 重启 ubuntu ,等待 grub 菜单的出现。
2.选择 recovery mode, 按 e 进入编辑界面。
3.将 ro recovery nomodeset 改为 rw single init=/bin/bash
4.按 ctrl+x或者F10 进入单用户模式，当前用户即为root。这时候可以修改文件。
5.修改好文件后，输入 reboot 进行重启即可。
```

## GDB

### arm-poky-linux-gdb（ubuntu x86）

```shell
source /opt/pancake-core-sdk/environment-setup-armv7ahf-neon-poky-linux-gnueabi
#arm-poky-linux-gdb
/opt/pancake-core-sdk/sysroots/x86_64-pokysdk-linux/usr/bin/arm-poky-linux/
```

### gdb（panel arm)

```bash
#cd /usr/src/linux-headers-5.3.0-62/include/linux 
#ln -s autoconf.h config.h
source /opt/pancake-core-sdk/environment-setup-armv7ahf-neon-poky-linux-gnueabi
cd gdb-9.2/
mkdir build
cd build
.././configure CFLAGS="-O3" CXXFLAGS="-O3" --target=arm-poky-linux-gnueabi --host=arm-linux --prefix=$SDKTARGETSYSROOT
sudo make -j 8
sudo make install
sudo cp gdb/gdb $SDKTARGETSYSROOT/bin
#copy file gdbserver & gdbreplay to panel's path (/usr/local/bin)
arm-poky-linux-gnueabi-strip gdb
```

### gdbserver（panel arm)

```bash
source /opt/pancake-core-sdk/environment-setup-armv7ahf-neon-poky-linux-gnueabi
cd gdb-9.2/gdb/gdbserver

./configure CFLAGS="-O3" CXXFLAGS="-O3"   --host=arm-linux --prefix=$SDKTARGETSYSROOT
sudo make -j 8

arm-poky-linux-gnueabi-strip gdbserver
arm-poky-linux-gnueabi-strip gdbreplay
sudo make install

#copy file gdbserver & gdbreplay to panel's path (/usr/local/bin)
cd /usr/local/bin
chmod +x gdbreplay
chmod +x gdbserver
```



```bash
sudo apt install net-tools
sudo apt-get install openssh-server

sudo passwd root  #修改密码

sudo vi /etc/apt/sources.list
sudo apt-get update
sudo apt-get -f install #修复受损软件包
```

## 修改国内源`sources.list`

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
```

## 编译环境

```bash
sudo apt install cmake git cmake-gui build-essential 
```

## opencv

```bash
sudo apt install libgtk2.0-dev libavcodec-dev libavformat-dev libjpeg-dev libswscale-dev libtiff5-dev pkg-config
```

## boost

```bash
sudo apt-get install libboost-all-dev 
```

## 安装Visual Studio Code

```bash
sudo add-apt-repository ppa:ubuntu-desktop/ubuntu-make
sudo apt-get update
sudo apt-get install ubuntu-make
umake ide visual-studio-code
```

## Qt5

```bash
sudo apt-get install  qt5-default qtcreator
```

## opencv for ubuntu

```bash
cd opencv-4.3.0
mkdir release && cd release

sudo cmake -D CMAKE_BUILD_TYPE=Release \
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

## opencv  for Arm

```bash
source /opt/pancake-core-sdk/environment-setup-armv7ahf-neon-poky-linux-gnueabi 
cd ~/opencv-4.3.0/platforms/linux
mkdir -p build_arm
cd build_arm
cmake -D CMAKE_BUILD_TYPE=Release -D CMAKE_INSTALL_PREFIX=$SDKTARGETSYSROOT -DCMAKE_TOOLCHAIN_FILE=../arm-gnueabi.toolchain.cmake ../../..
make -j 8
```

## googleTest

```bash
sudo apt-get install libssl-dev uuid-dev libmosquitto-dev libgtest-dev
sudo apt-get install mosquitto
sudo apt-get install lcov
```

## Vscode Remote-ssh

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

## Git Graph

## 配置git server（公钥）

add sshkey
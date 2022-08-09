# 编译环境

先说编译环境搭建，优先推荐docker，AOSP的源码里已经有使用docker的代码了：
> https://android.googlesource.com/platform/build/+/master/tools/docker

Dockerfile
```

FROM ubuntu:14.04
ARG userid
ARG groupid
ARG username
RUN apt-get update && apt-get install -y git-core gnupg flex bison gperf build-essential zip curl zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386 lib32ncurses5-dev x11proto-core-dev libx11-dev lib32z-dev ccache libgl1-mesa-dev libxml2-utils xsltproc unzip python openjdk-7-jdk
RUN curl -o jdk8.tgz https://android.googlesource.com/platform/prebuilts/jdk/jdk8/+archive/master.tar.gz \
 && tar -zxf jdk8.tgz linux-x86 \
 && mv linux-x86 /usr/lib/jvm/java-8-openjdk-amd64 \
 && rm -rf jdk8.tgz
RUN curl -o /usr/local/bin/repo https://storage.googleapis.com/git-repo-downloads/repo \
 && echo "d06f33115aea44e583c8669375b35aad397176a411de3461897444d247b6c220  /usr/local/bin/repo" | sha256sum --strict -c - \
 && chmod a+x /usr/local/bin/repo
RUN groupadd -g $groupid $username \
 && useradd -m -u $userid -g $groupid $username \
 && echo $username >/root/username \
 && echo "export USER="$username >>/home/$username/.gitconfig
COPY gitconfig /home/$username/.gitconfig
RUN chown $userid:$groupid /home/$username/.gitconfig
ENV HOME=/home/$username
ENV USER=$username
ENTRYPOINT chroot --userspec=$(cat /root/username):$(cat /root/username) / /bin/bash -i
```

构建镜像

```
# Copy your host gitconfig, or create a stripped down version
$ cp ~/.gitconfig gitconfig
$ docker build --build-arg userid=$(id -u) --build-arg groupid=$(id -g) --build-arg username=$(id -un) -t android-build-trusty .
```

编译

```
$ docker run -it --rm -v $ANDROID_BUILD_TOP:/src android-build-trusty
> cd /src; source build/envsetup.sh
> lunch aosp_arm-eng
> m -j50
```

# 源码

```
./repo init -u https://mirrors.tuna.tsinghua.edu.cn/git/lineageOS/LineageOS/android.git -b  lineage-16.0
```

# 提取专有blobs

这一步很重要！！

连接手机，进入源码目录脚本：`android/lineage/device/xiaomi/mido`
```
./extract-files.sh
```
# mido构建

```
croot
brunch mido
```

```
./repo sync

# 部分仓库例如Lineage_framework_base同步的时候会出现bundle错误
./repo sync --no-clone-bundle
```

# 红米相关

手机已退休，型号是Xiaomi Redmi Note 4 (mido)。

Recovery：”音量加“键+电源键

Bootloader：“音量减”键+电源键

查看手机是否解锁OEM:
Settings > Additional settings > Developer options > Mi Unlock status.

# 编完了怎么安装

下载TWRP:
> https://dl.twrp.me/mido/

连接手机，命令进入bootloader
```
adb reboot bootloader
```
刷入下载好的twrp镜像
```
fastboot flash recovery <recovery_filename>.img
```

使用TWRP刷入编译好的系统，选 “Advanced” => “ADB Sideload”

或者直接：
```
adb sideload LineageOS.zip
```
LineageOS.zip即编译好的系统文件。


# AOSP编译

以上是编译专属设备mido的流程，编译最新Anroid虚拟机版本参考：

> https://source.android.com/setup/build/initializing

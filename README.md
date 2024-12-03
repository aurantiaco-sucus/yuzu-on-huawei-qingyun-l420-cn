# 实验需求
* 华为擎云L420笔记本电脑
	* 在L410、W515等设备上可能也可行，但我没有可供实验的硬件
* 银河麒麟V10 2303桌面操作系统
	* 在统信UOS上可能也可行，但我的试用期已经结束了
	* 【更新】UOS 1060上不行，大概是因为某些库的版本太老
* yuzu的源代码
	* 可以从GitHub拉取
	* 【更新】Suyu也可以
* yuzu运行所需的各种资料
	* 请自行解决

# 替代用户空间
从yuzu的官方编译指导可以得知需要GCC v11+、CMake 3.15+才能完成编译，但遗憾的是银河麒麟V10的软件源中提供的版本无法达到这个要求，所以我们需要使用更加新的用户空间软件来完成这一点。

这里我使用[琥珀兼容环境（ACE）](https://gitee.com/amber-compatability-environment)的[书虫兼容模式](https://gitee.com/amber-compatability-environment/bookworm-compatibility-mode/tree/master)来达到这个要求，这个软件可以在[星火应用商店](https://spark-app.store/)下载安装。

# 配置编译环境
通过`bookworm-run`命令，可以进入ACE的Debian 12用户空间。

![（图片：bubblewrap、用户空间、ACE）](/qy2.png?raw=true "bubblewrap、用户空间、ACE")
使用apt包管理器安装编译依赖：
```
sudo apt-get install autoconf cmake g++-11 gcc-11 git glslang-tools libasound2 libboost-context-dev libglu1-mesa-dev libhidapi-dev libpulse-dev libtool libudev-dev libxcb-icccm4 libxcb-image0 libxcb-keysyms1 libxcb-render-util0 libxcb-xinerama0 libxcb-xkb1 libxext-dev libxkbcommon-x11-0 mesa-common-dev nasm ninja-build qtbase5-dev qtbase5-private-dev qtwebengine5-dev qtmultimedia5-dev libmbedtls-dev catch2 libfmt-dev liblz4-dev nlohmann-json3-dev libzstd-dev libssl-dev libavfilter-dev libavcodec-dev libswscale-dev
```
# 编译并配置yuzu
切换到yuzu源代码所在文件夹，配置编译CMake项目：
```
export VCPKG_FORCE_SYSTEM_BINARIES=1
mkdir build
cmake .. -GNinja -DYUZU_USE_BUNDLED_VCPKG=ON -DYUZU_TESTS=OFF
ninja
```
编译完成后，在项目build/bin文件夹可见yuzu二进制文件，但由于链接了来自Debian用户空间的库，这个yuzu只能在ACE内运行。此时启动yuzu，熟悉的”你没有密钥“弹窗应该会如约出现，这里请自行按照一般方法配置yuzu。
# 配置Vulkan支持
配置完成，细心的读者大略已经发现，yuzu的图像设置中Vulkan只有一个Mesa的软件渲染实现。若是用软件渲染运行游戏，那至少在这个8核心的移动处理器上是不好实现的。但好消息是麒麟9006c的Mali-G78 GPU具备Vulkan 1.2支持，已经足以运行目前版本的yuzu了。此时我们需要在容器内部利用Mali专有用户态驱动程序（libmali.so）来提供Vulkan支持。
![（图片：Vulkan支持）](/qy1.png?raw=true "Vulkan支持")
首先，我们需要修改ACE bubblewrap的运行脚本，将`/dev/mali0`这个由内核驱动提供的设备文件暴露到容器内。脚本位于`/opt/apps/cn.flamescion.bookworm-compatibility-mode/files/bin/bookworm-run`。笔者在实验的时候直接修改了这个文件，但实际上这样做大概不是最好的办法，个人建议还是复制这个文件到便于访问的地方，然后将这个位置添加到PATH。
![（图片：设备绑定）](/2024-01-30_19-45-51.png?raw=true "设备绑定")
如图所示，添加这个设备绑定。

这个操作完成之后，我们还需要实际把Mali专有用户态驱动程序也暴露到容器内。笔者采取了较为暴力的方法，直接将文件复制到了容器内（容器的根为`/opt/apps/cn.flamescion.bookworm-compatibility-mode/files/bookworm-env`）但读者也可以采取通过上面的配置文件（添加绑定）的方式来完成。具体需要提供的文件如下：
* /usr/lib/aarch64-linux-gnu/libmali.so
* /etc/vulkan/icd.d/mali_vulkan.json
* /usr/lib/libc_secshared.so

可以使用`vulkan-tools`包的`vulkaninfo`命令来查看Vulkan设备情况。若看到了Mali-G78的字样，那么大概算是成功了。
# 在Wayland下运行yuzu
如果读者已经尝试在yuzu使用”Mali-G78“ Vulkan设备运行软件，那么可能会遇到闪退（无法初始化Vulkan上下文）的情况，这大概是因为这个Vulkan实现只能在Wayland应用程序上使用。遗憾的是，笔者的知识水平尚不足以解释这种问题的原因，但这实际上并不影响我们使用yuzu的Vulkan后端。

（在build/bin文件夹）使用以下命令即可使得yuzu（作为一个Qt Widgets应用程序）在Wayland下运行。注意，这样会使得其缩放（在L420上是150%）失效。
```
QT_QPA_PLATFORM=wayland ./yuzu
```
此时读者应该已经可以正常运行软件了，但可能又会发现一个问题：
![（图片：陌生的SE）](/2024-01-30_20-07-53.png?raw=true "陌生的SE")
> 这个SQUARE ENIX标志怎么变蓝了

根据笔者推测，这个问题系蓝色和红色的通道位置发生了交换（也就是RGB和BGR的区别，读者中的数据科学家们可能比较熟悉），很遗憾，我没有找到根治这种问题的方法。也许全屏滤镜可能可以改善问题？

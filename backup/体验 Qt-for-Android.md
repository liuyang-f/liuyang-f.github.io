官方文档：https://doc.qt.io/qt-6/android.html

**目的：把 Qt 安装目录 Example 中的例子编译生成 APK，安装到 Android 平台运行起来。**

## 一、依赖安装
要把 Qt 项目编译成 APK，首先要解决的就是编译环境问题。它有两个部分内容，一个是如何配置 Android 的环境，另一个是如何把 Qt 与 Android 关联起来。
- Android 环境：使用 Android Studio 可以很方便的下载 `Android SDK`、`Android NDK`、`JDK` 依赖包。
  示例目录
  ```
  $ cd ~/Android/Sdk
  $ ls
  build-tools  cmake  cmdline-tools  emulator  licenses  ndk  platforms  platform-tools  skins  sources  system-images
  $ ls platforms
  android-31  android-35
  $ ls build-tools/
  31.0.0  34.0.0  35.0.0
  ```
- Qt 连接 Android：Qt 专门为编译 Android 提供一个模块，使用 Qt 的管理工具 `MaintenanceTool` 可以下载这个模块。
  示例目录
  ```
  $ cd ~/Qt/6.8.7
  $ ls
  android_arm64_v8a  android_armv7  android_x86  android_x86_64  gcc_64  sha1s.txt
  $ ls android_x86_64/
  bin  doc  include  jar  lib  libexec  metatypes  mkspecs  modules  phrasebooks  plugins  qml  qtinsight  src  translations
  ```
## 二、编译脚本
当 Qt 与 Android 环境依赖都安装以后，就可以进行进行编译，为了方便使用直接让 AI 生成一个脚本。
```
#!/bin/bash

# qt 文档
# https://doc.qt.io/qt-6/android-building-projects-from-commandline.html

# 配置路径
QT_ANDROID_BIN="/home/liuyang/Qt/6.8.7/android_x86_64/bin/qt-cmake"
ANDROID_SDK_ROOT="/home/liuyang/Android/Sdk"
ANDROID_NDK_ROOT="/home/liuyang/Android/Sdk/ndk/30.0.14904198"
BUILD_DIR="build-android"

# 尝试自动设置 JAVA_HOME (如果环境没设置)
if [ -z "$JAVA_HOME" ]; then
    # 尝试寻找系统默认 JDK 路径
    export JAVA_HOME="/home/liuyang/workspace/tool/android-studio/jbr"
fi

# 1. 清理并创建构建目录
if [ -d "$BUILD_DIR" ]; then
    echo "正在清理构建目录: $BUILD_DIR..."
    rm -rf "$BUILD_DIR"
fi
mkdir -p "$BUILD_DIR"
cd "$BUILD_DIR"

# 2. 配置项目
echo "正在配置项目..."
"$QT_ANDROID_BIN" \
    -DANDROID_SDK_ROOT="$ANDROID_SDK_ROOT" \
    -DANDROID_NDK_ROOT="$ANDROID_NDK_ROOT" \
    -S .. \
    -GNinja

# 3. 生成 Android 构建文件 (Gradle 项目)
echo "正在生成 Gradle 项目文件..."
# 运行此命令是为了让 Qt 自动生成 android-build 目录及其配置
# cmake --build . --target apk > /dev/null 2>&1
cmake --build . --target apk

# 4. 直接调用 gradlew 完成 APK 编译
echo "正在编译 APK (通过 gradlew)..."
cd android-build
./gradlew assembleDebug

# 5. 提示结果
if [ $? -eq 0 ]; then
    echo "------------------------------------------------"
    echo "编译成功！"
    echo "APK 位置: $BUILD_DIR/android-build/build/outputs/apk/debug/"
    echo "------------------------------------------------"
else
    echo "编译失败，请检查错误输出。"
    exit 1
fi
```

## 三、版本兼容
1、把 Qt 项目编译成 Android APK 的过程中，涉及到 Qt 版本、Android 版本以及依赖包的版本。
- Qt 版本决定 `AGP（Android Gradle Plugin）` 版本
- `AGP` 版本决定 `Gradle` 版本
- `AGP` 版本决定 `Android SDK`、`Android NDK`、`JDK` 版本

2、关于 `AGP` 与 `Gradle`
Qt 安装目录中内置了用于生成 android 项目的 Gradle 模板文件。
当使用 qt-cmake 配置并构建项目时，这些模板会被复制到构建目录中。
也就是在 Qt 的安装目录中可以看到 `AGP` 与 `Gradle` 的版本。
- `AGP（Android Gradle Plugin）` 版本
```
$ cd ~/Qt/6.8.7/android_x86_64
$ rg classpath
src/android/templates/build.gradle
9:        classpath 'com.android.tools.build:gradle:8.6.0'
```
- `Gradle` 版本
```
$ cd ~/Qt/6.8.7/android_x86_64
$ rg distributionUrl
src/3rdparty/gradle/gradle/wrapper/gradle-wrapper.properties
3:distributionUrl=https\://services.gradle.org/distributions/gradle-8.10-bin.zip
```

- 什么是 `Android Gradle Plugin`
它是 Google 开发的一组插件，运行在 Gradle 平台上。它赋予了 Gradle 处理 Android 特有任务的能力。
可以类比成 `cmake` 工具。

- 什么是 `Gradle`
Gradle 是一个开源的自动化构建工具，它不仅可以用于 Android，还可以用于 Java、C++ 或 Python 项目。
可以类比成 `make` 工具。

- 总结
如把 Gradle 看作是 Make 的进化版（支持网络依赖、更复杂的逻辑），那么 AGP 就是专门为这个进化版定制的 CMake 规则集。
没有 AGP，Gradle 就是个普通的 Java 构建工具，无法理解什么是 Android APK。
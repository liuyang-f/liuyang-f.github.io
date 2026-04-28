## 范例：camke加载QtCore和QtWidgets
- 1、使用`find_package`查找Qt5的组件QtCore和QtWidgets；
- 2、使用`target_link_libraries`链接`Qt5::Core`和`Qt5::Widgets`;
```
cmake_minimum_required(VERSION 3.10)

project(MyQtProject)

# 查找 Qt5（包名） 的特定组件 QtCore（模块名或者组件名） 和 QtWidgets（模块名或者组件名）；
# REQUIRED 表示这个包时必须的；
find_package(Qt5 REQUIRED COMPONENTS Core Widgets)

# 查找成功后，使用这些组件
add_executable(MyApp main.cpp)

# 链接 QtCore 和 QtWidgets 库
target_link_libraries(MyApp Qt5::Core Qt5::Widgets)

```

## find_package的三种查找模式
- 1、模块模式（Module mode）
  - 在 cmake 模块自己路径中，查找 `Find<PackageName>.cmake` 文件，文件由 camke 官方提供，不够灵活，使用较少。
  - 比如cmake提供的SQLite3库：`/usr/share/cmake-3.16/Modules/FindSQLite3.cmake`。
- 2、配置模式（Config mode）
  - 在通过 CMAKE_PREFIX_PATH 指定的路径中查找 `<PackageName>Config.cmake` 或 `<PackageName>-config.cmake` 文件，文件由包的开发者提供，比较通用，qt 使用这种方式；
- 3、源码构建模式（FetchContent redirection mode）
  - 不在查找包，而是直接下载依赖模块的源码，然后编译使用，不需要手动安装包，比较适合自动化。


## qt对cmake的查找支持
camke使用`Config mode`查找qt模块，说明qt肯定提供类似<PackageName>Config.cmake 或 <PackageName>-config.cmake之类的文件，果然在ubuntu的`/usr/lib/`目录发现如下：
```
/usr/lib/x86_64-linux-gnu/cmake/Qt5/Qt5Config.cmake
/usr/lib/x86_64-linux-gnu/cmake/Qt5Core/Qt5CoreConfig.cmake
/usr/lib/x86_64-linux-gnu/cmake/Qt5Widgets/Qt5WidgetsConfig.cmake
/usr/lib/x86_64-linux-gnu/cmake/Qt5Network/Qt5NetworkConfig.cmake
/usr/lib/x86_64-linux-gnu/cmake/Qt5OpenGL/Qt5OpenGLConfig.cmake
/usr/lib/x86_64-linux-gnu/cmake/Qt5Gui/Qt5GuiConfig.cmake
还有很多类似的文件 …… 
```
进一步探究，在 `Qt5CoreConfig.cmake` 文件中发现了 `set(Qt5Core_LIBRARIES Qt5::Core)` ；
同样的，在 `Qt5WidgetsConfig.cmake` 文件中发现了 `set(Qt5Widgets_LIBRARIES Qt5::Widgets)` ；
至此，和前面的 `target_link_libraries(MyApp Qt5::Core Qt5::Widgets)` 联系起来，原来能够链接到对应模块的库，都是因为 cmake 文件中有了相应的定义。


## 总结
使用 cmake 加载 qt 模块，需要使用 `find_package` 查找包名（Qt5）+ 模块名（Core、Widgets），以及使用 `target_link_libraries` 链接对应的库（Qt5::Core、Qt5::Widgets）。
以后，我们需要使用 qt 的其他模块，但是不知道模块名字和链接库的名字，就可以先找到 qt 库安装目录中对应的 "*Config.cmake" 文件，看看有哪些模块名字，并在 "*Config.cmake" 文件中找到类似 `Qt5Core_LIBRARIES` 和 `Qt5Core_LIBRARIES` 的定义，也就可以确定链接库的名字了。


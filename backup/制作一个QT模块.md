参考例子：https://github.com/emqx/qmqtt

有时候我们需要自定义一个模块，使用qmake、make、make install等工具编译安装后，开发者可以使用该模块，就像使用QT模块一样。
比如把qmqtt下载到本地，编译安装后，直接在pro文件中QT += qmqtt，包含头文件即可使用qmqtt的功能。
如果自己制作一个类似的库，需要注意哪些？

## 1、顶层目录文件`.qmake.conf`
  - 示例文件：https://github.com/emqx/qmqtt/blob/master/.qmake.conf
  - 该文件是顶层的隐藏文件，QT编译的时候会自动加载该文件，如果没有识别到该文件会对应的提示；在https://github.com/mburakov/qt5/blob/master/qtbase/mkspecs/features/qt_build_paths.prf 中有这样的处理：
  ```
    # Find the module's source root dir.
	isEmpty(_QMAKE_CONF_): error("Project has no top-level .qmake.conf file.")
  ```
  - 文件中的`load(qt_build_config)`
    - 为了让自定的模块能够像使用QT自己模块一样，需要把自定义模块按照QT的方式去编译，使用qt_build_config即可达到此目的，它会加载很多编译配置，就像是编译QT自己模块一样。
	- qt_build_config是QT编译时候加载的一个文件，地址：https://github.com/mburakov/qt5/blob/master/qtbase/mkspecs/features/qt_build_config.prf
## 2、顶层目录pro文件中的`load(qt_parts)`
  - 示例文件：https://github.com/emqx/qmqtt/blob/master/qmqtt.pro
  - qt_parts 也是QT编译时候加载的一个文件，地址：https://github.com/mburakov/qt5/blob/master/qtbase/mkspecs/features/qt_parts.prf
    - 它的作用，是检查当前目录中是否存在src、examples、demos、tests等目录，如果有，就指定对应目录中的pro文件。
	- qt_parts.prf中写的很清楚，一看就可以明白，类似片段如下：
	```
	exists($$_PRO_FILE_PWD_/src/src.pro) {
		sub_src.subdir = src
		sub_src.target = sub-src
		SUBDIRS += sub_src

			exists($$_PRO_FILE_PWD_/tools/tools.pro) {
				sub_tools.subdir = tools
				sub_tools.target = sub-tools
				sub_tools.depends = sub_src
				# conditional treatment happens on a case-by-case basis
				SUBDIRS += sub_tools
			}
	}
	```
## 3、子目录pro文件中的`load(qt_module)`
  - 示例文件：https://github.com/emqx/qmqtt/blob/master/src/mqtt/qmqtt.pro
  - qt_module 也是QT编译时候加载的一个文件，地址：https://github.com/mburakov/qt5/blob/master/qtbase/mkspecs/features/qt_module.prf
  - 一般来说，制作一个自定义的QT模块，肯定需要有so文件、.h文件，以及相关的模块描述文件，加载这个文件，编译后就可以生成所有的文件。
  
## 4、`make install`没有安装头文件
一般来说，在执行完`qmake && make`命令后，会在 build目录生成一个 include 子目录，include 目录下有一个以模块名字命名的子目录，它里边会有需要安装的所有头文件（.h）以及一个 headers.pri 文件。
headers.pri 文件记录了需要安装的头文件列表；但是自定义的模块，执行`make`以后，build 目录下没有对应的头文件，headers.pri 文件也没有记录头文件信息。
很自然的，执行`make install` 也没有把头文件安装到 Qt 的安装目录中，开发者也就无法导入头文件，不能正常使用这个自定义的模块。
基于上述这个问题，定位查找原因（找了好久，记录下），发现了两个关键点：

<img width="1741" height="298" alt="Image" src="https://github.com/user-attachments/assets/0df65491-1e26-4119-ac53-114342edc8ed" />

- 第一个关键点，没有调用 syncqt.pl 脚本

  对比 qmqtt 的编译输出，可以发现自定义的模块，在执行 `make` 的时候少了一个步骤：
  ```
  perl -w /QT_INSTALL/5.15.19/gcc_64/bin/syncqt.pl 
  ```
  接着，就出现了2个新的问题：（1）syncqt.pl 脚本作用是什么？（2）为什么没有调用这个脚本？
  - syncqt.pl 作用
    它是 QT 提供的一个 Perl 脚本，在构建 QT 模块时候扫描源码目录导入模块的头文件。
    既然没有调用该脚本，也就理所应该的没有导出模块的头文件。
    地址：https://github.com/GarageGames/Qt/blob/master/qt-5/qtbase/bin/syncqt.pl
  - 为啥没调用呢
    在 https://github.com/mburakov/qt5/blob/master/qtbase/mkspecs/features 查找 syncqt 关键字，果然有相关的处理 ，在 https://github.com/mburakov/qt5/blob/master/qtbase/mkspecs/features/qt_module_headers.prf 中有这样的处理：
    ```
    !build_pass:git_build {
      qtPrepareTool(QMAKE_SYNCQT, syncqt)
    }
    ```
    怀疑就是使用 qtPrepareTool 调用的 syncqt 脚本，继续查找 qtPrepareTool定义，在 https://github.com/mburakov/qt5/blob/master/qtbase/mkspecs/features/qt_functions.prf 中：
    ```
    defineTest(qtPrepareTool) {
      $$1 = $$eval(QT_TOOL.$${2}.binary)
      isEmpty($$1) {
          $$1 = $$[QT_HOST_BINS]/$$2
          exists($$eval($$1).pl) {
              $$1 = perl -w $$eval($$1).pl
          } else: contains(QMAKE_HOST.os, Windows) {
              $$1 = $$eval($$1).exe
          } else:contains(QMAKE_HOST.os, Darwin) {
              BUNDLENAME = $$eval($$1).app/Contents/MacOS/$$2
              exists($$BUNDLENAME) {
                  $$1 = $$BUNDLENAME
              }
          }
      }
      ……
    }
    ```
    已经可以看到 `perl -w $$eval($$1).pl` 的命令，说明就是通过这个函数调用的 syncqt 脚本。
    回到 qtPrepareTool 的调用处，需要满足条件 `!build_pass:git_build` ，才会调用。
    查找 `qtbase/mkspecs/features` 目录下的文件，发现很多地方都在使用条件 `!build_pass`，猜测这个条件一般编译都是满足的。
    重点放在第二个条件 `git_build`，在 https://github.com/mburakov/qt5/blob/master/qtbase/mkspecs/features/qt_build_paths.prf 中：
    ```
    exists($$MODULE_BASE_INDIR/.git): \
      CONFIG += git_build
    ```
    很明显，自定义的模块目录中需要有 .git 文件，才能满足条件，触发 syncqt.pl 脚本的调用。
    马上 `mkdir .git` 后，再次 `qmake && make`，果不其然，顺利执行了 syncqt.pl 脚本。
  
- 第二个关键点，有调用 syncqt.pl 脚本，但还是没有正常导出头文件到 include 目录中
  - 怀疑头文件查找目录不正确
    搜索相关资料，发现调用 syncqt.pl 脚本时候，会读取 sync.profile 文件，可以说它是 syncqt.pl 脚本的配置文件。
    sync.profile 文件决定了 syncqt.pl 脚本去哪个目录搜索头文件，然后导出，对应地址：https://github.com/emqx/qmqtt/blob/master/sync.profile
    各种折腾去设置 sync.profile 中的查找目录对应的字段，可以确定查找头目录的目录是正确的，但是执行 syncqt.pl 脚本 有些头文件始终无法导出。
    做了一个小测试，自己创建一些空的头文件，发现可以导入，此时可以确定和设置的查找目录无关了，怀疑和头文件的命名方式有关。

  - 怀疑和头文件命名有关
    怎么办，只有去看看 syncqt.pl 脚本源码，找找和查找头文件规则相关的处理。
    最终发现，syncqt.pl 脚本脚本有这样一行代码:
    ```
    my @headers = findFiles($subdir, "^[-a-z0-9_]*\\.h\$");
    ```

    这个可以发现，脚本只会查找开头为小写字母（a~z）、数字（0~9）、连字符（-）、下划线（_）等开头的头文件命名方式，对比自己头文件命名，果然头文件名字开头为大写的都没有导出，只有部分开头为小写的头文件有导出。
    有两个方法解决这个问题，一个是修改头文件命名，符合脚本的查找规则；二是直接修改脚本的查找逻辑；采用第二种方法，修改脚本的查找逻辑：
    ```
    # my @headers = findFiles($subdir, "^[-a-z0-9_]*\\.h\$");
    my @headers = findFiles($subdir, ".*\\.h\$");
    ```
    像这样修改后，可以导出所有的头文件，不再限定命名规则。
    再次执行该脚本，所有的.h文件都正常的导入到了 include 目录中了。
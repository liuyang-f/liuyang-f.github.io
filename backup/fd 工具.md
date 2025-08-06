fd 可以根据关键字，递归查找指定目录中的文件。
地址：https://github.com/cha0ran/fd
中文文档：https://github.com/cha0ran/fd-zh

1、查找文件
```
> fd netfl # 递归查找当前目录，只要文件名字中有 netfl，结果就会列出该文件
> fd -g libc.so # 递归查找当前目录，结果只会列出文件名和 libc.so 完全一样的文件
> fd '^x.*rc$' # 递归查找当前目录，使用正则限定，结果只会列出文件名字开头为 x，结尾是 rc的文件
> fd passwd /etc # 递归查找指定 目录 /etc
> fd -e md # 递归查找当前目录，只要文件扩展名为 md，结果就会列出该文件；用 -e（或 --extension）选项来完成
> fd -e rs mod # 递归查找当前目录，如果文件扩展名为 md，并且该文件名包含 mod，结果才会列出该文件

// 默认情况下，fd 只匹配每个文件的文件名。然而，使用 --full-path 或 -p 选项，你可以对全路径进行匹配
// 其中 “**” 是通配符，“.*” 是正则表达式
// * 匹配任意字符但不跨目录，** 可以跨目录匹配任意层级的子路径。
// src/**/*.cpp：匹配 src/xxx.cpp、src/a/b/c/main.cpp
// **/*.h：匹配所有子目录下的 .h 文件
> fd -p -g '**/.git/config' # 有 -g 参数，单引号中的字符串是 通配符表达式
> fd -p '.*/lesson-\d+/[a-z]+.(jpg|png)' #  没有 -g 参数，单引号中的字符串是 正则表达式
```

2、查找隐藏文件（-H）和 ignore 文件（-I）
fd 默认不搜索隐藏目录和隐藏文件；也不搜索符合 .gitignore 模式之一的文件夹。
```
> fd -HI netfl # 搜索所有文件和目录，包括隐藏文件和 .gitignore设置的文件
> fd -H netfl # 可以搜索隐藏文件
> fd -I netfl # 可以搜索 .gitignore设置的文件
```

3、不查找指定的文件（-E或 --exclude）
有时候，我们不想查找某些文件和目录，可以使用 -E（或 --exclude）选项来实现这一点。它需要一个任意的 glob 模式作为参数：
```
> fd -H -E .git …
> fd -E /mnt/external-drive …
> fd -E '*.bak' …
```
为了使排除模式永久化，可以创建一个 .fdignore 文件，具体参考文档。

4、递归列出文件，（类似于 ls -R）
```
> fd # 递归列出当前目录中文件
> fd . /etc # 递归列出指定目录 /etc 中文件
```

5、fd 查找文件后执行外部命令（-x/-X）
当使用 fd 查找到文件后，需要基于查找结果，执行 shell 命令，可以使用参数 -x/-X。
fd有2中方法执行外部命令：
- -x/--exec 选项为每个搜索结果运行一个外部命令（并行）
- -X/--exec-batch 选项启动一次外部命令，将所有搜索结果作为参数
例子如下：
```
fd -e zip -x unzip # 递归找到所有的压缩文件并解压：
fd -e h -e cpp -x clang-format -i # 找到所有的*.h和*.cpp文件，用clang-format -i将它们自动格式化
fd -g 'test_*.py' -X vim # 这里我们使用大写的 -X 来打开一个vim实例
fd … -X ls -lhd --color=always # 查看文件权限、所有者、文件大小等细节

// fd 提供了一个快捷方式。你可以使用 -l/--list-details 选项，以这种方式执行ls：
fd … -l

fd -e cpp -e cxx -e h -e hpp -X rg 'std::cout' # 把 fd 和 ripgrep（rg）结合起来
fd -tf -x md5sum > file_checksums.txt # 计算一个目录中每个单独文件的校验和

// 将所有 *.jpg 文件转换为 *.png 文件
// {} 是搜索结果的一个占位符。{.} 也是如此，但它没有文件扩展名。关于占位符语法的更多细节，需要查看文档。
// 如果你不包括占位符，fd 会自动在末尾添加一个 {}
fd -e jpg -x convert {} {.}.png 

// 删除文件
> fd -H '^\.DS_Store$' -tf -X rm
> fd -H '^\.DS_Store$' -tf -X rm -i
```
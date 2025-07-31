rg 使用文档：https://github.com/BurntSushi/ripgrep/blob/master/GUIDE.md
[rg](https://github.com/BurntSushi/ripgrep) 根据指定的正则字符串去递归搜索当前目录中的文件，并列出带有行号的结果。但会自动忽略一些文件：
（1）.gitignore文件、.ignore文件、.rgignore文件。使用参数 --no-ignore
（2）隐藏的文件和目录。使用参数 --hidden （简写为 -. ）
（3）二进制文件。使用参数--text （简写为 -a ）
（4）符号链接文件。使用 --follow （简写为 -L ）

## 1、递归搜索所有文件（rg提供的一种特殊便利的方法）
`$ rg fast -uuu`

## 2、在指定文件中搜索
`$ rg fast README.md`
`$ rg 'fast\w+' README.md`

## 3、递归目录搜索
`$ rg 'fn write\('`
`$ rg 'fn write\(' src`

## 4、搜索指定通配符 `-g` 的文件（注意-g后面使用的是单引号）
在所有的toml文件中搜索：
`$ rg clap -g '*.toml'`
搜索除了 `*.toml` 以外的所有文件
`$ rg clap -g '！*.toml'`

## 5、搜索 `rg` 可以识别的文件类型的文件
`$ rg 'fn run' -g '*.rs'`
`$ rg 'fn run' --type rust` （完全写法）
`$ rg 'fn run' -trust` （简洁写法）
使用 `-g` 参数，rg 只搜索指定后缀的文件；而使用 `--type` 可以搜索一系列的文件集合。
比如搜索 c 文件，包含c的源文件和头文件。
`$ rg 'int main' -g '*.{c,h}'`
达到同样效果，使用 `$ rg 'int main' -tc` 就会搜索c的源文件和头文件。
查看 rg 支持哪些文件类型，使用 `rg --type-list`
```
$ rg --type-list | rg '^make:'
make: *.mak, *.mk, GNUmakefile, Gnumakefile, Makefile, gnumakefile, makefile
```

## 6、不搜索指定文件类型
`$ rg clap --type-not rust`
简洁写法：`$ rg clap -Trust`
也就是说，使用 `-t` 搜索这种文件类型，使用 `-T` 不搜索这种文件类型。

## 7、添加自定义文件类型，让 rg 可以指定搜索
比如，你经常搜索 "web" 文件，这些文件由 JavaScript、HTML 和 CSS 组成：
`$ rg --type-add 'web:*.html' --type-add 'web:*.css' --type-add 'web:*.js' -tweb title`
简洁写法：
`$ rg --type-add 'web:*.{html,css,js}' -tweb title`
上述命令定义了一个新的类型， web ，对应于通配符 *.{html,css,js} 。然后使用 -tweb 应用新的过滤器并搜索模式 title 。如果你运行:
`$ rg --type-add 'web:*.{html,css,js}' --type-list`
那么你会在列表中看到你的 web 类型出现，即使它不是 ripgrep 的内置类型的一部分。

## 8、保存添加的自定义文件类型
一般来说，使用 `--type-add` 添加的自定义文件类型，只能在当前的终端下使用。
如何设置能够保证在其他终端也可以使用？
方法一：创建一个 shell 别名
`alias rg="rg --type-add 'web:*.{html,css,js}'"`
方法二：写入 [ripgrep 配置文件](https://github.com/BurntSushi/ripgrep/blob/master/GUIDE.md#configuration-file)中 

## 9、文件类型的特殊参数 all
使用 `rg --type all` 会搜索 `rg --type-list` 列出的所有文件类型（有扩展名）。
假如当前目录下有一个文件 `my-shell-script`，它引用了另一个文件 `my-shell-library.bash`。
由于文件`my-shell-script` 没有扩展名，即使使用 `rg --type all`，也不会去查找 文件`my-shell-script`，只会查找文件 `my-shell-library.bash`。
另外，使用 `rg --type-not all` 会搜索文件 `my-shell-script` ，但不会搜索文件 `my-shell-library.bash` 。
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

## 9、搜索二进制文件的3种模式
- 1. default mode（无参数）
  在非二进制文件中查找关键字。
  rg 会识别二进制文件，然后跳过二进制文件，在其他文件中搜索内容。
- 2. Binary mode（参数：`--binary`）
  可以搜索所有的文件；不会排除二进制文件的搜索。
  在二进制文件找到内容不会刷屏终端显示。
  rg 也会识别二进制文件，但是不会跳过二进制文件搜索内容。
- 3. Text mode（参数：`-a/--text`）
  可以搜索所有的文件；不会排除二进制文件的搜索。
  把所有文件当做文本处理，所以在二进制文件找到内容会刷屏终端显示。
  rg 会禁用二进制文件的监测，不会识别二进制文件。
  
## 10、使用预处理器（参数 `--pre`）
假如，我们需要搜索 PDF 文件。
虽然，PDF 文件是一种二进制格式；但是，PDF 中显示的文本可能不是以简单的连续 UTF-8 编码。
即使，我们传递 `-a/--text` 标志给 ripgrep，也无法根据关键字搜索到想要的结果。
比如搜索命令：
```
$ rg 'The Commentz-Walter algorithm' 1995-watson.pdf
$
```
结果为空的。使用参数 `--pre`可以解决这个问题。
简单来说，`--pre` 后面可以指定一个 shell 命令，当然也可以是一个 shell 脚本，通过这个明白把 PDF 转为文件执行搜索功能。
shell 脚本如下：
```
$ cat preprocess
#!/bin/sh
exec pdftotext - -
```
指定 shell 脚本后的搜索结果如下：
```
$ rg --pre ./preprocess 'The Commentz-Walter algorithm' 1995-watson.pdf
316:The Commentz-Walter algorithms : : : : : : : : : : : : : : :
7165:4.4 The Commentz-Walter algorithms
10062:in input string S , we obtain the Boyer-Moore algorithm. The Commentz-Walter algorithm
17218:The Commentz-Walter algorithm (and its variants) displayed more interesting behaviour,
17249:Aho-Corasick algorithms are used extensively. The Commentz-Walter algorithms are used
17297: The Commentz-Walter algorithms (CW). In all versions of the CW algorithms, a common program skeleton is used with di erent shift functions. The CW algorithms are
```
额外的好处是，这比其他专业的 PDF 搜索工具要快得多：
```
$ time rg --pre ./preprocess 'The Commentz-Walter algorithm' 1995-watson.pdf -c
6

real    0.697
user    0.684
sys     0.007
maxmem  16 MB
faults  0

$ time pdfgrep 'The Commentz-Walter algorithm' 1995-watson.pdf -c
6

real    1.336
user    1.310
sys     0.023
maxmem  16 MB
faults  0
```

## 11、使用参数（--pre-glob）过滤减少预处理器的开销
因为预处理器是针对每一个文件进行处理了，如果筛选指定文件类型后在使用预处理器，可以提高整体的执行速度。
```
$ time rg --pre pre-rg 'fn is_empty' -c
crates/globset/src/lib.rs:1
crates/matcher/src/lib.rs:2
crates/ignore/src/overrides.rs:1
crates/ignore/src/gitignore.rs:1
crates/ignore/src/types.rs:1

real    0.138
user    0.485
sys     0.209
maxmem  7 MB
faults  0

$ time rg --pre pre-rg --pre-glob '*.pdf' 'fn is_empty' -c
crates/globset/src/lib.rs:1
crates/ignore/src/types.rs:1
crates/ignore/src/gitignore.rs:1
crates/ignore/src/overrides.rs:1
crates/matcher/src/lib.rs:2

real    0.008
user    0.010
sys     0.002
maxmem  7 MB
faults  0
```

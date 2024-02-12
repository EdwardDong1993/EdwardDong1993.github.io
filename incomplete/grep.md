## 1 简介

**grep（global regular expression）** 命令，用于查找文件或标准输入中符合条件的字符串或正则表达式，默认的正则表达式为 BRE（basic regular expression），用法如下：

```
grep [OPTION]... PATTERN [FILE]...
```

当没有 FILE 参数时，将从标准输入中读取数据。

## 2 选项

**正则表达式选项**

|选项|说明|
|:----:|----|
|-E|PATTERN 是 ERE（extended regular expression）|
|-G|PATTERN 是 BRE（basic regular expression）|
|-P|PATTERN 是 Perl regular expression|
|-e|需要匹配的 PATTERN|
|-f <file>|从文件中获取 PATTERN|
|-i|忽略大小写|
|-w|整个单词进行匹配|
|-x|整行进行匹配|

**输出选项**

|选项|说明|
|:----:|----|
|-m <n>|匹配 n 个后停止|
|-b|显示输出行的字节偏移|
|-n|显示行号|
|-H|显示该行所属的文件名称|
|-h|不显示该行所属的文件名称|
|-o|只输出匹配的 pattern，不整行输出|
|-q|不输出任何信息|
|-a|二进制文件|
|-d <action>|对文件夹进行操作，操作包括 'read'，'recurse'，'skip'|
|-r|递递归查找子目录中的文件|
|-L|只输出不匹配的文件名|
|-l|只输出匹配的文件名|
|-c|输出匹配的行数计数|

**上下文选项**

|选项|说明|
|:----:|----|
|-B <n>|除了输出符合 pattern 的那一行之外，输出该行之前 n 行的内容|
|-A <n>|除了输出符合 pattern 的那一行之外，输出该行之后 n 行的内容|
|-C <n>|除了输出符合 pattern 的那一行之外，输出该行前后 n 行的内容|

**其它选项**

|选项|说明|
|:----:|----|
|-s|不显示错误信息|
|-v|反向查找，只打印不匹配的行|

## 3 一些总结

#### 3.1 BRE 和 ERE

BRE 和 ERE 的区别在于元字符，在 BRE 中如果想要元字符表示特殊的含义，需要把它们转义；反之在 ERE 中如果要元字符表示普通字符，需要把它们转义。

#### 3.2 -e 选项

好像没有什么特别的用处，不加该选项也可以正常搜索正则表达式，该选项的用处可能在于可以匹配多个表达式：

```
grep -e "expression1" -e "expression1" -e "expression3" file
```
#### 3.3 -o 选项

一般输出都会输出整行，然后对 PATTERN 匹配的字符进行高亮，而添加 `-o` 选项后，就只输出 PATTERN 匹配的字符。

不添加 `-o` 选项时搜索结果如下：
```
$ grep --help |grep "Example"
Example: grep -i 'hello world' menu.h main.c
```

添加 `-o` 选项时搜索结果如下：
```
$ grep --help |grep -o "Example"
Example
```

#### 3.4 快捷搜索

搜索某时间段内的日志:
```
grep -E "09:30:[0-9]{2}" file
grep -E "09:(30|31|32):" file
```

搜索某关键字下一行信息：
```
grep -A 1 "expression" file |grep -v "expression"
```

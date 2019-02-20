**YAML多行字符串**

为YAML多行字符串找到正确的语法

> 英文原文：[YAML Multiline](https://yaml-multiline.info/)

YAML支持两种格式的字符串：块标量（block scalar）和流标量（flow scalar）格式。（像数字或字符串这种基本值在YAML中称为标量，与之对应的是像数组或对象这种复杂类型。）块标量可以更好地控制它们的解释方式，而流标量具有更多有限制的转义支持。

# 块标量

一个块标量头部有三个部分：

**块样式指示器（Block Style Indicator）**：块样式指示块内的换行符应如何表现。如果希望将它们保留为换行符，应使用字面样式（literal style），用一个管道符号（|）表示。如果你希望它们被空格替换，应使用折叠样式（folded style），用一个右尖括号（>）表示。（在使用折叠样式时，想要得到一个换行符，需要输入两个换行符才会保留一个空白行。带有额外缩进的行不会被折叠。）

**块裁剪[^1]指示器（Block Chomping Indicator）**：裁剪指示器控制字符串末尾的换行符应该如何处理。默认情况下，会缩减（clip）为字符串末尾的一个换行符。如果要删除所有换行符，需在样式指示器后添加一个减号（-）来删除（strip）它们。缩减（clip）和删除（strip）都会忽略块的末尾实际上有多少换行符；在样式指示器后添加一个加号（+）将保留所有换行符。

[^1]: 译者注，chomping一词未见标准翻译，暂译为“裁剪”。

**缩进指示器（Indentation Indicator）**：通常，你用以缩进一个块的空格数可以从其首行自行推测出。但如果该块的首行以额外的空格开始，那么你应该就需要一个块缩进指示器了。在这种情况下，在头部的末尾简单地添加用来缩进的空格数（1~9）即可。

## 示例[^2]

[^2]: 译者注，原文示例是可动态配置的，如想尝试各配置的不同效果，请查看原文。

**块标量样式**（[？](https://yaml.org/spec/1.2/spec.html#id2795688)）

* 用空格替换换行符（折叠）
* 保留换行符（字面）

**块裁剪**（[？](https://yaml.org/spec/1.2/spec.html#id2794534)）

* 结尾单换行符（缩减）
* 结尾无换行符（删除）
* 结尾所有换行符（保留）

**缩进**

缩进空格数：2

包括缩进指示器



YAML

```yaml
example: >2\n
··Several lines of text,\n
··with some "quotes" of various 'types',\n
··and also a blank line:\n
··\n
··plus another line at the end.\n
··\n
··\n
```

结果

```
Several lines of text, with some "quotes" of various 'types', and also a blank line:\n
plus another line at the end.\n
```

# 流标量

## 单引号

YAML

```yaml
example: 'Several lines of text,\n
containing ''single quotes''. Escapes (like \n) don''t do anything.\n
\n
Newlines can be added by leaving a blank line.\n
··Leading whitespace on lines is ignored.'\n
```

结果

```
Several lines of text, containing 'single quotes'. Escapes (like \n) don't do anything.\n
Newlines can be added by leaving a blank line. Leading whitespace on lines is ignored.
```

## 双引号

YAML

```yaml
example: "Several lines of text,\n
containing \"double quotes\". Escapes (like \\n) work.\nIn addition,\n
newlines can be esc\\n
aped to prevent them from being converted to a space.\n
\n
Newlines can also be added by leaving a blank line.\n
··Leading whitespace on lines is ignored."\n
```

结果

```
Several lines of text, containing "double quotes". Escapes (like \n) work.\n
In addition, newlines can be escaped to prevent them from being converted to a space.\n
Newlines can also be added by leaving a blank line. Leading whitespace on lines is ignored.\n
```

## 普通

YAML

```yaml
example: Several lines of text,\n
··with some "quotes" of various 'types'.\n
··Escapes (like \n) don't do anything.\n
··\n
··Newlines can be added by leaving a blank line.\n
····Additional leading whitespace is ignored.\n
```

结果

```
Several lines of text, with some "quotes" of various 'types'. Escapes (like \n) don't do anything.\n
Newlines can be added by leaving a blank line. Additional leading whitespace is ignored.
```

注意：普通流标量对“:”和“#”两个字符的使用有诸多限制。它们可以出现在字符串中，但是，“:”不能出现在空格或换行符之前，“#”不能出现在空格或换行符之后，否则将导致语法错误。如果你需要使用这些字符，最好使用引号样式中的一种。
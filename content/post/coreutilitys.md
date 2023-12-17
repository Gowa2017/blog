---
title: Linux Coreutilitys
date: 2017-12-07 14:12:19
updated: 2017-12-07 15:16:19
categories:
  - [Linux/Unix]
---
# tr 字符翻译

	tr [option] ... set1 [set2]
 -c 不在 `c`后跟集合字符
 -d 删除
 -s 压缩
 -t 替换
# cut 只显示指定内容

	cut option ... file
 -b 按字节
 -c 按字符
 -d 替换分域符（默认TAB），与f配合使用
 -f 指定显示字段(field)
 -n 配合-b，不分割多字节字符
 --complement 取补
 -s 不显示不含分域符的行
 -output-delimited=STRING 输出分域符，默认与输入相同
对于范围的指定
 N 第N个
 N- N到行尾
 N-M N到M
 -M 行首到第M
# expr 计算表达式

	expr EXPRESSION
	expr OPTION
表达式可能的值是：

	ARG1 | ARG2
	ARG1 & ARG2
	ARG1 < ARG2
	ARG1 <= ARG2
	ARG1 = ARG2
	ARG1 != ARG2
	ARG1 >= ARG2
	ARG1 > ARG2
	ARG1 + ARG2
	ARG1 - ARG2
	ARG1 * ARG2
	ARG1 / ARG2
	ARG1 % ARG2
	STRING : REGEXP/match STRING REGEXP
	substr STRING POS LENGTH
	index STRING CHARS
	length STRING
	+ TOKEN
	( EXPRESSION )


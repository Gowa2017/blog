---
title: Basic RegEx与Extend RegEx
toc: true
date: 2016-11-24 14:17:21
tags: [Shell, RegEx]
categories: 
  - [Linux/Unix]
---
当我们使用sed/awk/grep的时候，必然会用到的就是`正则表达式`，对与我们选取与操作字符串非常的有用。但却有两种比较常见的表达式，`BRE(Basic RegEx)和ERE(Extent RegEx)`，这其中确有些不同。
<!--more-->
# 不同
在GNU上来说，`BRE`与`ERE`的不同在于几个元字符的意义 __不同__ ：`?`, `+`, `()`, `{}`, `|`。  
`BRE` 要求在这几个字符加上 `\` 才能表示特殊意义，否则就匹配其自己，而在`ERE`中，匹配其自己需要用`\`转义。
>`\|` 是一个GNU扩展，`BRE`并不提供这个功能。  

Linux中，`SED`(通过`-r/-E`支持`ERE`)、`VIM`、`GREP`（通过`-E`选项支持`ERE`)使用的是`BRE`，`GAWK`使用的是`ERE`。
# BRE
`CHAR`	匹配指定字符。  
`.`	匹配任意一个字符。  
`^`	匹配行首非空字符。  
`$`	匹配行尾非空字符。  
`[LIST] [^LIST]`	匹配/不匹配指定字符。  
`(REGEXP1 | REGEXP2)`	匹配REGEXP1或REGEXP2。 __GNU扩展__  
`REGEXP1REGEXP2`	匹配两个连续的RegEx。  
`\n`	换行符  
`\CHAR`	转义字符。`$, *, ., \, ^`  
`*`	匹配任意次数。  
`\+`	匹配>=1次。  
`\?`	匹配0或1次。  
`\{I\}`	匹配I次。  
`\{I,J\}`	匹配I<=N<=J次。  
`\{I,\}`	匹配I <= N次。  
`\(REGEXP\)`	分组，用\[1-9]引用。  
`\DIGIT`	匹配上面的分组  


# ERE

`.`	匹配任意一个字符。  
`CHAR`	匹配指定字符。  
`^`	匹配行首非空字符。  
`$`	匹配行尾非空字符。  
`[LIST] [^LIST]`	匹配/不匹配指定字符。  
`(REGEXP1 | REGEXP2)`	匹配REGEXP1或REGEXP2。 __GNU扩展__  
`REGEXP1REGEXP2`	匹配两个连续的RegEx。  
`\n`	换行符  
`\CHAR`	转义字符。`$, *, ., \, ^`  
`*`	匹配任意次数。  
__`+`	匹配>=1次。__  
__`?`	匹配0或1次。__  
__`{I}`	匹配I次。__  
__`{I,J}`	匹配I<=N<=J次。__  
__`{I,}`	匹配I <= N次。__  
__`{,J}`	匹配N <= J次。 GNU特性__  
`\(REGEXP\)`	分组，用\[1-9]引用。  
`\DIGIT`	匹配上面的分组  

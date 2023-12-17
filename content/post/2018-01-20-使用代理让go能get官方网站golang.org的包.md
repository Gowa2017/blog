---
title: 使用代理让go能get官方网站golang.org的包
categories:
  - Golang
date: 2018-01-20 21:36:12
updated: 2018-01-20 21:36:12
tags:
  - Golang
---
开始学golang变成的时候，需要设置一下IDE，用的是vim，然后配合官方推荐的vim-go插件，然后这个插件需要装很多附加的包，有的是从golang.org网站下载的，很多时候都无法下载，国内就是这么蛋疼。后面使用了代理进行了配置才能下载。
<!--more-->
# 代理
首先需要开启一个代理，本地用的是ss代理，开启了http代理服务器，然后进行设置。设置环境变量：

ss客户端用的是 Shadowsocks-NG。

	export http_proxy=https://localhost:1087
	export https_proxy=https://localhost:1087
然后使用` go get `就可以正常下载了。

```
$go get -v golang.org/x/tools/guru
Fetching https://golang.org/x/tools/guru?go-get=1
Parsing meta tags from https://golang.org/x/tools/guru?go-get=1 (status code 200)
get "golang.org/x/tools/guru": found meta tag get.metaImport{Prefix:"golang.org/x/tools", VCS:"git", RepoRoot:"https://go.googlesource.com/tools"} at https://golang.org/x/tools/guru?go-get=1
get "golang.org/x/tools/guru": verifying non-authoritative meta tag
Fetching https://golang.org/x/tools?go-get=1
Parsing meta tags from https://golang.org/x/tools?go-get=1 (status code 200)
golang.org/x/tools (download)
package golang.org/x/tools/guru: cannot find package "golang.org/x/tools/guru" in any of:
	/usr/local/go/src/golang.org/x/tools/guru (from $GOROOT)
	/Users/wodediannao/go/src/golang.org/x/tools/guru (from $GOPATH)
wodedianaodeAir:src shouzheng.zhang$ go get -v  golang.org/x/tools/cmd/guru
golang.org/x/tools/go/buildutil
golang.org/x/tools/go/types/typeutil
golang.org/x/tools/go/ast/astutil
golang.org/x/tools/cmd/guru/serial
golang.org/x/tools/container/intsets
golang.org/x/tools/refactor/importgraph
golang.org/x/tools/go/ssa
golang.org/x/tools/go/loader
golang.org/x/tools/go/callgraph
golang.org/x/tools/go/ssa/ssautil
golang.org/x/tools/go/pointer
golang.org/x/tools/go/callgraph/static
golang.org/x/tools/cmd/guru
```

大功告成

# 1  Go 环境变量

使用 `go env` 命令来获得完整的列表以及每个变量的简要说明:

```
GO111MODULE=""
GOARCH="amd64"
GOBIN=""
GOCACHE="/Users/cxy/Library/Caches/go-build"
GOENV="/Users/cxy/Library/Application Support/go/env"
GOEXE=""
GOFLAGS=""
GOHOSTARCH="amd64"
GOHOSTOS="darwin"
GOINSECURE=""
GOMODCACHE="/Users/cxy/go/pkg/mod"
GONOPROXY=""
GONOSUMDB=""
GOOS="darwin"
GOPATH="/Users/cxy/go"
GOPRIVATE=""
GOPROXY="https://proxy.golang.org,direct"
GOROOT="/usr/local/go"
GOSUMDB="sum.golang.org"
GOTMPDIR=""
GOTOOLDIR="/usr/local/go/pkg/tool/darwin_amd64"
GCCGO="gccgo"
AR="ar"
CC="clang"
CXX="clang++"
CGO_ENABLED="1"
GOMOD=""
CGO_CFLAGS="-g -O2"
CGO_CPPFLAGS=""
CGO_CXXFLAGS="-g -O2"
CGO_FFLAGS="-g -O2"
CGO_LDFLAGS="-g -O2"
PKG_CONFIG="pkg-config"
GOGCCFLAGS="-fPIC -m64 -pthread -fno-caret-diagnostics -Qunused-arguments -fmessage-length=0 -fdebug-prefix-map=/var/folders/0k/vxgrgh5j1_1dxhgd5781pl6r0000gn/T/go-build735409318=/tmp/go-build -gno-record-gcc-switches -fno-common"
```

## 2.1 Go 的工作空间

从2009推出 Go 语言后，在 Go 开发者如何组织代码和依赖项方面发生了多次变化。

对于现代 Go 开发，规则很简单：你可以按照合适的方式自由地组织你的项目。

Go 仍然希望设置一个工作空间 GOPATH，用于存放使用 `go install` 安装的第三方 Go 工具。默认的 GOPATH 是 `$HOME/go`，第三方工具的源码保存在 `$GOPATH/src`(此处仍需进一步学习)，编译后的二进制文件存储在 `$GOPATH/bin`。

建议显式定义 GOPATH，并将 `$GOPATH/bin`目录放在 `PATH` 中：

```
export GOPATH=$HOME/go
export PATH=$PATH:$GOPATH/bin
```

##### GOPROXY: 设置下载代理

```
go env -w GO111MODULE=on
go env -w GOPROXY=https://goproxy.cn,direct // 中国区
```



### 关于 GO111MODULE

todo

#### 附录A: 过时的环境变量

1. __GOROOT__ 环境变量已经不再需要。

# 2 Go 命令集

###### go run: 运行 Go 脚本

`go run hello.go` 用来把 Go 程序当做一个脚本来处理，它会把编译的结果放到一个临时目录，并从临时目录执行二进制文档，然后在程序结束时删除二进制文件。

##### go build: 编译 Go 程序，产生二进制文件

`go build -o hello_world hello.go`

##### go install 需要继续学习，尤其是关于

# 3 代码格式化

__Go 指定的了强制的标准化格式：编写操作源代码的工具变得非常容易，带来了工具的优势__

`go format` 可以自动格式化代码。

Go 语言和 C、Java一样，在每条末尾都添加一个分号，但是不需要开发者显式添加，Go 编译器会按照 Effective Go 中的规则添加分号。

如果换行之前的最后一个token是以下任何一种，则词法分析器会在该标记后插入一个分号：

1. 标识符，包括 int ,float64等词
2. 基本字面量，如数字和字符串常量
3. 以下标记之一： break, continue, fallthrough, return, ++, --, ), }

由上面的规则可知，`{` 必须放在行尾，而不能另起一行，比如如下代码：

```
func main() 
{
	fmt.Println("Hello, world!")
}
```

根据分号插入原则，编译器将把它转化为:

```
func main();
{
	fmt.Println("Hello, world!");
};
```


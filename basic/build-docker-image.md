# 为 Go 程序构建 Docker 镜像

:thinking: 为什么要使用 Docker？为程序提供一致的运行环境，不依赖特定主机

> [《Docker 从入门到实践》- 为什么要使用 Docker？](https://vuepress.mirror.docker-practice.com/introduction/why/)

## 示例程序

hello web

```go
package main

import (
	"fmt"
	"net/http"
	"time"
)

func greet(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello World! %s", time.Now())
}

func main() {
	http.HandleFunc("/", greet)
	http.ListenAndServe(":8080", nil)
}
```

## 基础版本

在同一目录创建 `Dockerfile`，其实大部分语法就是字面意思，不需要太多解释，详细的文档可以看 [Dockerfile reference](https://docs.docker.com/engine/reference/builder/)

```dockerfile
FROM golang:1.18

# 可选，访问官方 proxy 慢的话
ENV GOPROXY=https://goproxy.cn

WORKDIR /go/src/app
COPY . .

RUN go build -o /go/bin/app

CMD [ "/go/bin/app" ]
```

根据 `Dockerfile` 里的指令构建镜像：

- `-t`: 指定镜像的 tag
- `.`: 表示在当前目录寻找 `Dockerfile`

```bash
docker build -t maolonglong/helloweb .
```

运行容器，然后浏览器访问 <http://localhost:8080>

```bash
docker run -d -p 8080:8080 maolonglong/helloweb
```

## 多阶段构建

既然是基础版本肯定有一些缺陷，构建出的镜像足足有九百多兆，里面还留着 Go 语言环境（编译型语言**静态编译**后可以单独运行），并且 golang 镜像默认是基于 Debian 制作的，`apt` 之类的软件太过臃肿

> 虽然可以通过 tag 指定其他更小的发行版例如 `golang:1.18-alpine`，但是依然会有三百多兆

先用带有 Go 环境的镜像编译出二进制文件，然后复制到一个干净的 `alpine`，最终成果 9.94MB :tada:

```dockerfile
FROM golang:1.18-alpine3.15 AS build-env

ENV GO111MODULE=on \
    CGO_ENABLED=0 \
    GOPROXY=https://goproxy.cn

WORKDIR /go/src/app
COPY . .

RUN go build -o /go/bin/app -trimpath -ldflags "-s -w"

FROM alpine:3.15
COPY --from=build-env /go/bin/app /
CMD [ "/app" ]
```

## 更小更安全

> ["Distroless" Container Images](https://github.com/GoogleContainerTools/distroless)

`gcr.io` 是谷歌的 container registry，得 fq

```dockerfile
FROM golang:1.18-alpine3.15 AS build-env

ENV GO111MODULE=on \
    CGO_ENABLED=0 \
    GOPROXY=https://proxy.golang.org

WORKDIR /go/src/app
COPY . .

RUN go build -o /go/bin/app -trimpath -ldflags "-s -w"

FROM gcr.io/distroless/static:nonroot
COPY --from=build-env /go/bin/app /
CMD [ "/app" ]
```

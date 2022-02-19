### dockerfile

https://www.codeleading.com/article/66893503567/ aphine镜像不可用 引用了cgo



安装docker-compoe

curl -L  https://github.com/docker/compose/releases/download/1.29.2/docker-compose-Linux-x86_64  -o docker-compose

```dockerfile
FROM  AS builder
WORKDIR /build
COPY . /build

RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.tuna.tsinghua.edu.cn/g' /etc/apk/repositories && apk add --no-cache git
RUN git config --global url."https://账号:密码@域名".insteadOf "https://域名"
RUN go env -w GO111MODULE=on && go env -w GOPROXY=https://goproxy.cn,direct && go env -w GOPRIVATE=域名 && go env -w GOSUMDB=off && go mod tidy
RUN CGO_ENABLED=0 GOOS=linux go build -o main main.go

FROM alpine:latest
WORKDIR /app

RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.tuna.tsinghua.edu.cn/g' /etc/apk/repositories && apk add --no-cache curl net-tools
COPY --from=builder /build/main  /usr/bin/main

CMD ["main"]
```





```dockerfile
FROM centos As builder
#FROM unitedwardrobe/golang-librdkafka:alpine-3.8-golang-1.11.10-librdkafka-1.0.0

RUN git config --global credential.helper store
RUN echo "http://账号:密码@域名" >> ~/.git-credentials
RUN go env -w GOPRIVATE="域名/*"
ENV GOPROXY https://goproxy.cn

WORKDIR /project
ADD . .
RUN go build ./main.go

FROM centos:7.7.1908
WORKDIR /app/go/src
RUN mkdir -p /app/go/src
COPY --from=builder /project/main /app/go/src
COPY /config/default_template /app/go/src/config/default_template
CMD ["/app/go/src/main", "--profile", "test_k8s", "--configServerUrl", "", "--configBranch", "master"]
```



https://lessisbetter.site/2020/11/10/dockerfile-arg arg用法
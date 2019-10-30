---
title: "Go in Docker"
date: 2019-10-30T12:11:10-05:00
draft: false
category: 
tags:
- go
- github
- docker
summary: "Deploy an application to docker"
---
# How do deploy an application in docker?

Start with a simple Go application that echos back a response.

{{< highlight sh >}}
curl http://<host>/echo/ping
#=> pong
{{< /highlight >}}

Code for the server is hosted at :
```
git clone http://github.com/bilal-bhatti/echo.git
```

Build docker image
{{< highlight sh >}}
cd echo
env GOOS=linux GOARCH=amd64 go build -o ./dist/echo-linux-amd64
docker build -t $GIT_USER/echo/linux-amd64 .
{{< /highlight >}}

Run image
{{< highlight sh >}}
docker run -d -p 8888:8888 $GIT_USER/echo/linux-amd64 -listen ":8888"
{{< /highlight >}}

Test

{{< highlight sh >}}
curl localhost:8888/echo/ping
#=> pong
{{< /highlight >}}

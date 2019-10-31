---
title: "Github Package Registry"
date: 2019-10-12T12:11:48-05:00
draft: false
category: 
tags:
- go
- docker
summary: "Code and Docker images in one place"
---

# Code and Docker images in one place

GitHub launched a package registry service, so let's give it a go. I'll use the [echo](https://github.com/bilal-bhatti/echo.git) server to go through the steps below.

Refer to [Go in Docker]({{< ref "posts/go-in-docker.md" >}} ) for creating a docker image.

Sign up for GitHub Package Registry [HERE](https://github.com/features/package-registry). Then create an authentication token and use it as your password for the Docker login command

```
https://github.com/features/package-registry
```

Add registry to Docker
{{< highlight sh >}}
docker login docker.pkg.github.com --username $GIT_USER
{{< /highlight >}}

{{< highlight sh >}}
docker build -t $GIT_USER/echo/linux-amd64:1.0.0 .
docker tag $GIT_USER/echo/linux-amd64 docker.pkg.github.com/$GIT_USER/echo/linux-amd64:1.0.0
docker push docker.pkg.github.com/$GIT_USER/echo/linux-amd64:1.0.0
{{< /highlight >}}


Run your newly built and published image with:
{{< highlight sh >}}
docker run -d -p 8888:8888 docker.pkg.github.com/$GIT_USER/echo/linux-amd64:1.0.0 -listen ":8888"
{{< /highlight >}}

---
title: "Github Package Registry"
date: 2019-10-30T12:11:48-05:00
draft: false
category: 
tags:
- go
- docker
summary: "Code and Docker images in one place"
---

# Code and Docker images in one place

Sign up for GitHub Package Registry [HERE](https://github.com/features/package-registry). Then create an authentication token and use it as your password for the Docker login command

```
https://github.com/features/package-registry
```

Add registry to Docker
{{< highlight sh >}}
docker login docker.pkg.github.com --username $GIT_USER
{{< /highlight >}}

Build your image and publish it

Dockerfile
{{< highlight docker >}}
FROM scratch
ADD "./echo" "/"
ENTRYPOINT ["/echo"]
{{< /highlight >}}

{{< highlight sh >}}
docker build -t $GIT_USER/$APP/$ARCH:$VERSION .
docker tag $GIT_USER/$APP/$ARCH docker.pkg.github.com/$GIT_USER/$APP/$ARCH:$VERSION
docker push docker.pkg.github.com/$GIT_USER/$APP/$ARCH:$VERSION
{{< /highlight >}}


Run your newly built and published image with:
{{< highlight sh >}}
docker run -d docker.pkg.github.com/$GIT_USER/echo/linux-amd64:1.0.0
{{< /highlight >}}

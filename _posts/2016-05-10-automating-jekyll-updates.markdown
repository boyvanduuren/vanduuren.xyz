---
title:  "Automating my Jekyll updates"
date:   2016-05-10 17:55:05+02:00
description: Serving Jekyll from docker + auto-updates
---

Like most other people I know, I don't like doing stuff by hand. So when I had to rebuild and reupload
this website twice two days ago because of some small errors, I got cranky.
There are a lot of nice docker images that serve Jekyll sites, like [jekyll/jekyll][1], but
none of them seemed to be run on ARM. So no Raspberry PI :(.

This was a nice opportunity to build a nice Docker container, which resulted in [boyvanduuren/jekyll-arm][2].
[Armbuild/debian][3] was used as a base for this image. I tried out Alpine first but ran into some issues
compiling [runit][4]. As it was about 0400am at that time I didn't want to solve that problem and switched
to debian.

Anyway, there are three things you need to use the [jekyll-arm][2] image:

* an ARM machine (tested on an RPI2b)
* an SSL certificate for the domain
* a compiled Jekyll site in a git repository
<br>  
When the container starts it'll pull the repository, start serving its `_site` directory with
nginx, and check for updates every 5 minutes.

If you want to try it yourself:

```bash
docker pull boyvanduuren/jekyll-arm
docker run -d   \
    -e GIT_URL=https://github.com/boyvanduuren/vanduuren.xyz.git \
    -e GIT_BRANCH=live \
    -e DOMAIN=vanduuren.xyz \
    -v /etc/ssl/private/:/etc/ssl/private/ \
    -p 80:80 \
    -p 443:443 \
    boyvanduuren/docker-jekyll
```

Replace the environment variables by your own, of course.

There are a few things I might add in the future, like toggling SSL on/off, but for my use-case
that's not really important.

So that was a fun little project. Learned a bit about Docker. Future plans might involve a pipeline
using [shippable][3] or [codeship][4]. Shippable looks better because you can provide your own
build hosts, which might be required if I want to build ARM images.

"Why all this for a small Jekyll site?", you might think. Well.. because why not?


[1]: https://hub.docker.com/r/jekyll/jekyll/
[2]: https://hub.docker.com/r/boyvanduuren/jekyll-arm/
[3]: https://hub.docker.com/r/armbuild/debian/
[4]: http://smarden.org/runit/
[5]: https://app.shippable.com/
[6]: https://codeship.com/

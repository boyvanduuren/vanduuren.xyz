---
title:		"Golang: Vendoring and Interface Confusion"
date:   	2017-02-12 16:05:16+01:00
description:	Isues when using a library that vendors dependencies
---

The last few days I have been working on a small tool that enables you to easily
run tests against your code hosted on Github on your own machines.  The thing
is, there are tons of tools and providers that allow you to do this, but they'll
start costing money as soon as you want to use them on private repositories.

So, inspired by Gitlab's excellent [gitlab-runner][gitlab-runner], I started
working on [octorunner][github-octorunner], a tool that implements a few of the
features gitlab-runner has, like running tests on private repositories in
ephemeral docker containers, triggered by push events from Github.  As a bonus
this was a nice chance to do a project in Go.

Anyway, after getting to a point where octorunner actually did something
useful, I wanted to add some tests to octorunner.  This is where a bit of
headache started. See, I use the [Docker client API][github-dockerapi] and they
vendor (all?) their external dependencies.

I had not come across vendoring in Go before, but since Go 1.5 - although
experimental - it supports vendoring. It was enabled by default in 1.6.
What happens when importing external packages into your project is that the
compiler will first search for a package in your project's `vendor`
directory, only after that will it search in `$GOPATH/src`.

Because the Docker client library vendored the `context` package - among others
- I ran into issues when I wanted to define an interface that the Docker client
was an implementation of.  The exact compilation error I ran into was as
follows:

```go
ImageExists: *client.Client does not implement ImageLister (wrong type for
ImageList method) have
ImageList("github.com/docker/docker/vendor/golang.org/x/net/context".Context,
types.ImageListOptions) ([]types.ImageSummary, error) want
ImageList("context".Context, types.ImageListOptions) ([]types.ImageSummary,
error)
```

So what's happening here exactly?
Let's take a look at a bit of code that reproduces this exact error:

```go
package main

import (
	"golang.org/x/net/context"
	"github.com/docker/docker/api/types"
	"github.com/docker/docker/client"
)

type ImageLister interface {
	ImageList(ctx context.Context, options types.ImageListOptions) ([]types.ImageSummary, error)
}

func main() {
	cli, _ := client.NewEnvClient()
	ImageExists(cli)
}

func ImageExists(lister ImageLister) {
	return
}
```

In the above example we're defining an `ImageLister` interface containing a
single method. The argument causing the issue is `ctx context.Context`. 
The `client.Client` struct returned by `client.NewEnvClient()` *should* implement the
`ImageLister` interface. However, the `Context` interface that's being used as
type in `Client.ImageList(...)` uses the Docker client library's vendored
`Context`, which is imported from
`$GOPATH/src/github.com/docker/docker/vendor/golang.org/x/net/context`.

`Context` in my program is being imported from
`$GOPATH/src/golang.org/x/net/context`. In the Docker library it's being
imported from the previously mentioned path. This causes the compiler to treat
them as two different interfaces, with the error as a result.

The thing about this that confuse(s/d) me, is that in Go all [interfaces are
implemented implicitly][go-implicit]. So I thought: every object that implements
Docker's vendored version of `Context`, also implements the local `Context`, so
this should work.  I might be missing something here.. I'll need to explore this
a bit further.

In the end I had to properly vendor the Docker client library, causing both the
(vendored) Docker client library's, and my own `golang.org/x/net/context`
imports to be imported from
`$GOPATH/src/github.com/boyvanduuren/octorunner/vendor/golang.org/x/net/context`,
resulting in them being exactly the same interface.

__*TL;DR:*__ When using library `L` that vendors other libraries, and you want
to create a common interface that uses an interface from one of those vendored
libraries, you'll likely run into the same problems as I did, and you'll have to
vendor library `L` in your project.

[gitlab-runner]:	https://gitlab.com/gitlab-org/gitlab-ci-multi-runner
[github-octorunner]:	https://github.com/boyvanduuren/octorunner
[github-dockerapi]:	https://github.com/docker/docker/tree/master/client
[go-implicit]:		https://tour.golang.org/methods/10

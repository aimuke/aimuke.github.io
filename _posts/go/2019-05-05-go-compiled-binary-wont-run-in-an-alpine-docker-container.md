---
title: "alpine 容器中无法运行go程序"
tags: [go, runtime, alpine, glibc]
---

# Go-compiled binary won't run in an alpine docker container on Ubuntu host

By default, if using the net package a build will likely produce a binary with some dynamic linking, e.g. to libc. You can inspect dynamically vs. statically link by viewing the result of ldd output.bin

There are two solutions I've come across:

- Disable CGO, via CGO_ENABLED=0
- Force the use of the Go implementation of net dependencies, netgo via go build -tags netgo -a -v, this is implemented for a certain platforms

From https://golang.org/doc/go1.2:

> The net package requires cgo by default because the host operating system must in general mediate network call setup. On some systems, though, it is possible to use the network without cgo, and useful to do so, for instance to avoid dynamic linking. The new build tag netgo (off by default) allows the construction of a net package in pure Go on those systems where it is possible.

The above assumes that the only CGO dependency is the standard library's net package.

# 参考文献
- [Go-compiled binary won't run in an alpine docker container on Ubuntu host](https://stackoverflow.com/questions/36279253/go-compiled-binary-wont-run-in-an-alpine-docker-container-on-ubuntu-host)


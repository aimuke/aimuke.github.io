---
title: "通过go-zookeeper了解zk的异步通知模式"
tags: [zookeeper, go, go-zookeeper]
---

**2016年04月29日 / elian **

本文通过使用` github.com/samuel/go-zookeeper/zk` 包，了解一下`zookeeper`的异步通知模式

通过 `zk.Connect` 可取得连接对象 (`*zk.Conn` 对象) `conn`，`zookeeper` 的相关 api 都在 `conn`中

```go
conn, _, err := zk.Connect([]string{"127.0.0.1:2181"}, time.Second)
if err != nil {
    panic(err)
}
```
`zk`包中声明了以下五种事件类型, 分别对应的api是 ：
- `EventNodeCreated`：节点创建事件，需要watch一个不存在的节点，当节点被创建时触发，此watch通过conn.ExistsW(path string)设置
- `EventNodeDeleted`：节点删除事件，需要watch一个已存在的节点，当节点被移除时触发，此watch通过conn.ExistsW(path string)设置
- `EventNodeDataChanged`：节点数据变化事件，此watch通过conn.GetW(path string) 以及 conn.ExistsW(path string) 设置，
- `EventNodeChildrenChanged`：子节点改变事件（数量改变），此watch通过conn.ChildrenW(path string)设置， 当path 下面增删子节点时触发（修改path下的子节点的内容时，不会触发通知）。
- `EventNoWatching`：watch移除事件，服务端出于某些原因不再为客户端watch节点时触发。

实例代码如下:
```go
package main
import (
    "log"
    "sync"
    "time"
    "github.com/samuel/go-zookeeper/zk"
)
var wg *sync.WaitGroup
func main() {
    conn, _, err := zk.Connect([]string{"127.0.0.1:2181"}, time.Second)
    if err != nil {
        panic(err)
    }
    defer conn.Close()
    //zk 包没有提供rmr命令，只能递归删除了
    if b, _, _ := conn.Exists("/demo"); b {
        log.Println("exists /demo")
        paths, _, _ := conn.Children("/demo")
        for _, p := range paths {
            conn.Delete("/demo/"+p, -1)
        }
        err = conn.Delete("/demo", -1)
        if err != nil {
            log.Println(err)
        } else {
            log.Println("delete /demo")
        }
    }
    wg = &sync.WaitGroup{}
    watchDemoNode("/demo", conn)
    wg.Wait()
}
func watchDemoNode(path string, conn *zk.Conn) {
    wg.Add(1)
    //创建
    watchNodeCreated(path, conn)
    //改值
    go watchNodeDataChange(path, conn)
    //子节点变化「增删」
    go watchChildrenChanged(path, conn)
    //删除节点
    watchNodeDeleted(path, conn)
    wg.Done()
}
func watchNodeCreated(path string, conn *zk.Conn) {
    log.Println("watchNodeCreated")
    for {
        _, _, ch, _ := conn.ExistsW(path)
        e := <-ch
        log.Println("ExistsW:", e.Type, "Event:", e)
        if e.Type == zk.EventNodeCreated {
            log.Println("NodeCreated ")
            return
        }
    }
}
func watchNodeDeleted(path string, conn *zk.Conn) {
    log.Println("watchNodeDeleted")
    for {
        _, _, ch, _ := conn.ExistsW(path)
        e := <-ch
        log.Println("ExistsW:", e.Type, "Event:", e)
        if e.Type == zk.EventNodeDeleted {
            log.Println("NodeDeleted ")
            return
        }
    }
}
func watchNodeDataChange(path string, conn *zk.Conn) {
    for {
        _, _, ch, _ := conn.GetW(path)
        e := <-ch
        log.Println("GetW('"+path+"'):", e.Type, "Event:", e)
    }
}
func watchChildrenChanged(path string, conn *zk.Conn) {
    for {
        _, _, ch, _ := conn.ChildrenW(path)
        e := <-ch
        log.Println("ChildrenW:", e.Type, "Event:", e)
    }
}
```
# 参考文献
- [原文地址](http://blog.elian.xyz/article/go-zookeeper-watcher)

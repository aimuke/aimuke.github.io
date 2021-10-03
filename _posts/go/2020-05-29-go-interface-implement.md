---
title: "interface 源码分析"
tags: [go, interface]
---

# interface 底层结构

根据 `interface` 是否包含有 `method` ，底层实现上用两种 `struct` 来表示： `iface` 和 `eface` 。

`eface` 表示不含 `method` 的 `interface` 结构，或者叫 `empty interface`。对于 Golang 中的大部分数据类型都可以抽象出来 `_type` 结构，同时针对不同的类型还会有一些其他信息。

```go
    type eface struct {
        _type *_type
        data  unsafe.Pointer
    }
    
    type _type struct {
        size       uintptr // type size
        ptrdata    uintptr // size of memory prefix holding all pointers
        hash       uint32  // hash of type; avoids computation in hash tables
        tflag      tflag   // extra type information flags
        align      uint8   // alignment of variable with this type
        fieldalign uint8   // alignment of struct field with this type
        kind       uint8   // enumeration for C
        alg        *typeAlg  // algorithm table
        gcdata    *byte    // garbage collection data
        str       nameOff  // string form
        ptrToThis typeOff  // type for pointer to this type, may be zero
    }
```

`iface` 表示 `non-empty interface` 的底层实现。相比于 empty interface，non-empty 要包含一些 method。method 的具体实现存放在 `itab.fun` 变量里。

```go
    type iface struct {
        tab  *itab
        data unsafe.Pointer
    }
    
    // layout of Itab known to compilers
    // allocated in non-garbage-collected memory
    // Needs to be in sync with
    // ../cmd/compile/internal/gc/reflect.go:/^func.dumptypestructs.
    type itab struct {
        inter  *interfacetype
        _type  *_type
        link   *itab
        bad    int32
        inhash int32      // has this itab been added to hash?
        fun    [1]uintptr // variable sized
    }
```

试想一下，如果 `interface` 包含多个 `method` ，这里只有一个 `fun` 变量怎么存呢？

其实，通过反编译汇编是可以看出的，中间过程编译器将根据我们的转换目标类型的 empty interface 还是 non-empty interface，来对原数据类型进行转换（转换成 `<*_type, unsafe.Pointer>` 或者 `<*itab, unsafe.Pointer>`）。这里对于 `struct` 满不满足 `interface` 的类型要求（也就是 `struct` 是否实现了 `interface` 的所有 `method` ），是由编译器来检测的。


# iface 之 itab

`iface` 结构中最重要的是 `itab` 结构。 `itab` 可以理解为 `pair<interface type, concrete type>` 。当然 `itab` 里面还包含一些其他信息，比如 interface 里面包含的 method 的具体实现。下面细说。`itab` 的结构如下。

```go
    type itab struct {
        inter  *interfacetype  // 接口定义的类型信息
        _type  *_type          // 接口实际指向值的类型信息
        link   *itab
        bad    int32
        inhash int32      // has this itab been added to hash?
        fun    [1]uintptr // 接口方法实现列表，即函数地址列表，按字典序排序
    }
```

其中 `interfacetype` 包含了一些关于 interface 本身的信息，比如 `package path`，包含的 `method`。上面提到的 `iface` 和 `eface` 是数据类型（`built-in` 和 `type-define`）转换成 interface 之后的实体的 `struct` 结构，而这里的 `interfacetype` 是我们定义 `interface` 时候的一种抽象表示。

```go
    type interfacetype struct {
        typ     _type
        pkgpath name
        mhdr    []imethod // 接口方法声明列表，按字典序排序
    }
    
    type imethod struct {   //这里的 method 只是一种函数声明的抽象，比如  func Print() error
        name nameOff
        ityp typeOff
    }
```

`_type` 表示 concrete type。`fun` 表示的 interface 里面的 `method` 的具体实现。比如 interface type 包含了 method A, B，则通过 `fun` 就可以找到这两个 method 的具体实现。

为了提高查找效率， `runtime` 中实现 `(interface_type, concrete_type) -> itab` (包含具体方法实现地址等信息) 的 `hash` 表。并不是每次接口赋值都要去检查一次对象是否符合接口要求，而是只在第一次生成 `itab` 信息，之后通过 `hash` 表即可找到 `itab` 信息。

# 接口赋值

```go
type MyInterface interface {
   Print()
}
type MyStruct struct{}
func (ms MyStruct) Print() {}
func main() {
   a := 1
   b := "str"
   c := MyStruct{}
   var i1 interface{} = a
   var i2 interface{} = b
   var i3 MyInterface = c
   var i4 interface{} = i3
   var i5 = i4.(MyInterface)
   fmt.Println(i1, i2, i3, i4, i5)
}
```

对于接口间的赋值，将 `iface` 赋给 `eface` 比较简单，直接提取 `eface` 的 `interfacetype` 和 `data` 赋给 `iface` 即可。而反过来，则需要使用接口断言，接口断言通过 `assertE2I`, `assertI2I` 等函数来完成，这类 `assert` 函数根据使用方调用方式有两个版本

```go
i5 := i4.(MyInterface)      // call conv.assertE2I
i5, ok := i4.(MyInterface)  // call conv.AssertE2I2
```

下面看一下几个常用的 `conv` 和 `assert` 函数实现:

```go
func convT2E(t *_type, elem unsafe.Pointer) (e eface) {
    if raceenabled {
        raceReadObjectPC(t, elem, getcallerpc(), funcPC(convT2E))
    }
    if msanenabled {
        msanread(elem, t.size)
    }
    x := mallocgc(t.size, t, true)
    // TODO: We allocate a zeroed object only to overwrite it with actual data.
    // Figure out how to avoid zeroing. Also below in convT2Eslice, convT2I, convT2Islice.
    typedmemmove(t, x, elem)
    e._type = t
    e.data = x
    return
}

func convT2I(tab *itab, elem unsafe.Pointer) (i iface) {
    t := tab._type
    if raceenabled {
        raceReadObjectPC(t, elem, getcallerpc(), funcPC(convT2I))
    }
    if msanenabled {
        msanread(elem, t.size)
    }
    x := mallocgc(t.size, t, true)
    typedmemmove(t, x, elem)
    i.tab = tab
    i.data = x
    return
}

func assertE2I(inter *interfacetype, e eface) (r iface) {
    t := e._type
    if t == nil {
        // explicit conversions require non-nil interface value.
        panic(&TypeAssertionError{"","", inter.typ.string(), ""})
    }
    r.tab = getitab(inter, t, false)
    r.data = e.data
    return
}
```

在 `assertE2I` 中，我们看到了 `getitab` 函数，即 `i5=i4.(MyInterface)` 中，会去判断 `i4` 的 `concretetype(MyStruct)` 是否满足 `MyInterface` 的 `interfacetype` ，由于前面我们执行过 `var i3 MyInterface = c`，因此 `hash[itabhash(MyInterface, MyStruct)]` 已经存在 `itab`，所以无需再次检查接口是否满足，从 `hash` 表中取出 `itab` 即可 (里面针对接口的各个方法实现地址都已经初始化完成)。

而在 go1.10 中，有一些优化:

`convT2x` 针对简单类型 (如 `int32,string,slice`) 进行特例化优化 (避免 `typedmemmove`):

```
convT2E16, convT2I16
convT2E32, convT2I32
convT2E64, convT2I64
convT2Estring, convT2Istring
convT2Eslice, convT2Islice
convT2Enoptr, convT2Inoptr
```
优化了剩余对 `convT2I` 的调用:

由于 `itab` 由编译器生成, 可以直接由编译器将 `itab` 和 `elem` 直接赋给 `iface` 的 `tab` 和 `data` 字段，避免函数调用和 `typedmemmove`

对接口的构造和转换本质上是对 `object` 的 `type` 和 `data` 两个字段的操作，对空接口 `eface` 来说，只需将 `type` 和 `data` 提取并填入即可，而对于非空接口 `iface` 构造和断言，需要判断 `object` 或 `eface` 是否满足接口定义，并生成对应的 `itab` (包含接口类型，object类型，object接口实现方法地址等信息)，每个已初始化的 `iface` 都有 `itab` 字段，该字段的生成是通过 `hash` 表优化的，以及对于每个 `interfacetype <-> concrettype` 对，只需要生成一次 `itab` ，之后从 `hash` 表中取就可以了。由于编译器知晓 `itab` 的内存布局，因此在将 `iface` 赋给 `eface` 的时候可以避免函数调用，直接将 `iface.itab.typ` 赋给 `eface.typ`。

# interface的内存布局

了解 interface 的内存结构是非常有必要的，只有了解了这一点，我们才能进一步分析诸如类型断言等情况的效率问题。先看一个例子：

```go
    type Stringer interface {
        String() string
    }
    
    type Binary uint64
    
    func (i Binary) String() string {
        return strconv.Uitob64(i.Get(), 2)
    }
    
    func (i Binary) Get() uint64 {
        return uint64(i)
    }
    
    func main() {
        b := Binary{}
        s := Stringer(b)
        fmt.Print(s.String())
    }
```

根据上面 interface 的源码实现，可以知道，interface 在内存上实际由两个成员组成，如下图， `tab` 指向虚表， `data` 则指向实际引用的数据。虚表描绘了实际的类型信息及该接口所需要的方法集

![Uploading interface内存布局_731644.png]

观察 `itab` 的结构，首先是描述 `type` 信息的一些元数据，然后是满足 `Stringger` 接口的函数指针列表（注意，这里不是实际类型 `Binary` 的函数指针集哦）。因此我们如果通过接口进行函数调用，实际的操作其实就是 `s.tab->fun0`。

golang为每种类型创建了一个方法集，接口的虚表是在运行时专门生成的。可能细心的同学能够发现为什么要在运行时生成虚表。因为太多了，每一种接口类型和所有满足其接口的实体类型的组合就是其可能的虚表数量，实际上其中的大部分是不需要的，因此golang选择在运行时生成它，例如，当例子中当首次遇见 `s := Stringer(b)` 这样的语句时，golang会生成 `Stringer` 接口对应于 `Binary` 类型的虚表，并将其缓存。

理解了golang的内存结构，再来分析诸如类型断言等情况的效率问题就很容易了，当判定一种类型是否满足某个接口时，golang使用类型的方法集和接口所需要的方法集进行匹配，如果类型的方法集完全包含接口的方法集，则可认为该类型满足该接口。例如某类型有 `m` 个方法，某接口有 `n` 个方法，则很容易知道这种判定的时间复杂度为 `O(mXn)` ，不过可以使用预先排序的方式进行优化，实际的时间复杂度为 `O(m+n)` 。

# interface 与 nil 的比较

引用公司内部同事的讨论议题，觉得之前自己也没有理解明白，为此，单独罗列出来，例子是最好的说明，如下

```go
package main

import (
	"fmt"
	"reflect"
)

type State struct{}

func testnil1(a, b interface{}) bool {
	return a == b
}

func testnil2(a *State, b interface{}) bool {
	return a == b
}

func testnil3(a interface{}) bool {
	return a == nil
}

func testnil4(a *State) bool {
	return a == nil
}

func testnil5(a interface{}) bool {
	v := reflect.ValueOf(a)
	return !v.IsValid() || v.IsNil()
}

func main() {
	var a *State
	fmt.Println(testnil1(a, nil))
	fmt.Println(testnil2(a, nil))
	fmt.Println(testnil3(a))
	fmt.Println(testnil4(a))
	fmt.Println(testnil5(a))
}
```

返回结果如下

```
false
false
false
true
true
```

解释

`interface{}` 类型的比较是同时满足 type 和 data 都相等才相等。

- `testnil1` 参数 `a` 传入时转换成 `interface{}`, 其 type 为 `*State`,与 nil 的type 不相等
- `testnil2` 函数中比较时，`a *State` 也是先转换成 interface{} 在比较，只是转换时间点的区别。故与 `testnil1` 相同，也是不相等的。
- `testnil3` 也是 `a *State` 转换成 `interface{}` 后 type 为空，所以不相等
- `testnil4` 这里的比较不是 `interface{}` 的比较，而是指针的比较，指针的空值为 `nil` 所以相等
- `testnil5` 由于变量 `a *State` 的值为 `nil`, 所以为 true 

# 附录

## 生成itab

假定，`interface` 定义了 `ni` 个方法，构造类型实现 `nt` 个方法，

常规匹配构造类型是否实现全部 `ni` 个方法需要两层遍历，复杂度为 `O(ni*nt)`

这样在初始化 `itab.fun` 或类型断言匹配是效率会比较低。

Go 设计时也考虑了这个问题，把复杂度降低为 `O(ni+nt)`

这也是使用 hashtable 的原因之一：

首先 interface 的函数定义列表 `itab.inter.mhdr` 和构造类型的函数列表 `itab.fun` 都是按函数名排好序的
这样第一次 `itab` 初始化时，判定构造类型是否实现函数列表可以 `O(ni+nt)` 内遍历完成
然后用开放地址探测法更新到 `itab` 中，查询时也可以用同样的方式定位到此 `itab` 是否存在。

为了提高查找效率，runtime中实现`(interface_type, concrete_type) -> itab`(包含具体方法实现地址等信息)的hash表, 可以看到，并不是每次接口赋值都要去检查一次对象是否符合接口要求，而是只在第一次生成 `itab` 信息，之后通过 `hash` 表即可找到 `itab` 信息。

```go
// runtime/iface.go
const (
   hashSize = 1009
)

var (
   ifaceLock mutex // lock for accessing hash
   hash      [hashSize]*itab
)
// 简单的Hash算法
func itabhash(inter *interfacetype, typ *_type) uint32 {
   h := inter.typ.hash
   h += 17 * typ.hash
   return h % hashSize
}

// 根据interface_type和concrete_type获取或生成itab信息
func getitab(inter *interfacetype, typ *_type, canfail bool) *itab {
   ...
    // 算出hash key
   h := itabhash(inter, typ)


   var m *itab
   ...
           // 遍历hash slot链表
      for m = (*itab)(atomic.Loadp(unsafe.Pointer(&hash[h]))); m != nil; m = m.link {
         // 如果在hash表中找到则返回
         if m.inter == inter && m._type == typ {
            if m.bad {
               if !canfail {
                  additab(m, locked != 0, false)
               }
               m = nil
            }
            ...
            return m
         }
      }
   }
    // 如果没有找到，则尝试生成itab(会检查是否满足接口)
   m = (*itab)(persistentalloc(unsafe.Sizeof(itab{})+uintptr(len(inter.mhdr)-1)*sys.PtrSize, 0, &memstats.other_sys))
   m.inter = inter
   m._type = typ
   additab(m, true, canfail)
   if m.bad {
      return nil
   }
   return m
}

// 检查concrete_type是否符合interface_type 并且创建对应的itab结构体 将其放到hash表中
func additab(m *itab, locked, canfail bool) {
   inter := m.inter
   typ := m._type
   x := typ.uncommon()

   ni := len(inter.mhdr)
   nt := int(x.mcount)
   xmhdr := (*[1 << 16]method)(add(unsafe.Pointer(x), uintptr(x.moff)))[:nt:nt]
   j := 0
   for k := 0; k < ni; k++ {
      i := &inter.mhdr[k]
      itype := inter.typ.typeOff(i.ityp)
      name := inter.typ.nameOff(i.name)
      iname := name.name()
      ipkg := name.pkgPath()
      if ipkg == "" {
         ipkg = inter.pkgpath.name()
      }
      for ; j < nt; j++ {
         t := &xmhdr[j]
         tname := typ.nameOff(t.name)
         // 检查方法名字是否一致
         if typ.typeOff(t.mtyp) == itype && tname.name() == iname {
            pkgPath := tname.pkgPath()
            if pkgPath == "" {
               pkgPath = typ.nameOff(x.pkgpath).name()
            }
            // 是否导出或在同一个包
            if tname.isExported() || pkgPath == ipkg {
               if m != nil {
                    // 获取函数地址，并加入到itab.fun数组中
                  ifn := typ.textOff(t.ifn)
                  *(*unsafe.Pointer)(add(unsafe.Pointer(&m.fun[0]), uintptr(k)*sys.PtrSize)) = ifn
               }
               goto nextimethod
            }
         }
      }
      // didn't find method
      if !canfail {
         if locked {
            unlock(&ifaceLock)
         }
         panic(&TypeAssertionError{"", typ.string(), inter.typ.string(), iname})
      }
      m.bad = true
      break
   nextimethod:
   }
   if !locked {
      throw("invalid itab locking")
   }
   // 加到Hash Slot链表中
   h := itabhash(inter, typ)
   m.link = hash[h]
   m.inhash = true
   atomicstorep(unsafe.Pointer(&hash[h]), unsafe.Pointer(m))
}
```

# References

- [Golang interface接口深入理解, 吴德宝AllenWu](https://juejin.im/post/5a6873fd518825734501b3c5)

- [Go Interface 从理解到深入](https://zhaolion.com/post/golang/upgrade/interface/)

- [Go Interface 实现](https://wudaijun.com/2018/01/go-interface-implement/)

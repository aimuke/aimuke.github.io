# 循环控制Goto、Break、Continue


循环控制语句可以控制循环体内语句的执行过程。GO 语言支持以下几种循环控制语句：`Goto` 、 `Break` 、 `Continue`

1. 三个语句都可以配合标签(`label`)使用
2. 标签名区分大小写，定以后若不使用会造成编译错误
3. `continue` 、 `break` 配合标签(`label`)可用于多层循环跳出
4. `goto` 是调整执行位置，与 `continue` 、 `break` 配合标签(`label`)的结果并不相同

# goto

`goto` 语句 将控制转移到被标记的语句。

Go 语言的 `goto` 语句可以无条件地转移到过程中指定的行。

`goto` 语句通常与条件语句配合使用。可用来实现条件转移， 构成循环，跳出循环体等功能。但是，在结构化程序设计中一般不主张使用 `goto` 语句， 以免造成程序流程的混乱，使理解和调试程序都产生困难。

## 语法

`goto` 语法格式如下：

```go
goto label;
..
.
label: statement;
```

Golang支持在函数内 `goto` 跳转。标签名区分大小写，未使用标签引发错误。

```go
func main() {
    var i int
    for {
        println(i)
        i++
        if i > 2 { goto BREAK }
    }
BREAK:
    println("break")
EXIT:                 // Error: label EXIT defined and not used
}
```

## goto 实例

```go
package main

import "fmt"

func main() {
   /* 定义局部变量 */
   var a int = 10

   /* 循环 */
   LOOP: for a < 20 {
      if a == 15 {
         /* 跳过迭代 */
         a = a + 1
         goto LOOP
      }
      fmt.Printf("a的值为 : %d\n", a)
      a++
   }  
}
```

Golang支持在函数内 `goto` 跳转。 `goto` 语句与标签之间不能有变量声明。否则编译错误。

```go
package main

import "fmt"

func main() {
	fmt.Println("start")
	goto Loop

	var i int = 1
Loop:
	fmt.Println(i)
	fmt.Println("end")
}
```

编译错误：

```
./main.go:7:7: goto Loop jumps over declaration of i at ./main.go:9:6
```

以上实例执行结果为：
```
a的值为 : 10
a的值为 : 11
a的值为 : 12
a的值为 : 13
a的值为 : 14
a的值为 : 16
a的值为 : 17
a的值为 : 18
a的值为 : 19
```

# 控制语句

## break

`break` 语句 经常用于中断当前 `for` 循环或跳出 `switch` 语句

Go 语言中 `break` 语句用于以下两方面：
1. 用于循环语句中跳出循环，并开始执行循环之后的语句。
2. break在switch（开关语句）中在执行一条case后跳出语句的作用。

实例:

```go
package main

import "fmt"

func main() {
   /* 定义局部变量 */
   var a int = 10

   /* for 循环 */
   for a < 20 {
      fmt.Printf("a 的值为 : %d\n", a)
      a++
      if a > 15 {
         /* 使用 break 语句跳出循环 */
         break
      }
   }
}
```

以上实例执行结果为：

```go
a 的值为 : 10
a 的值为 : 11
a 的值为 : 12
a 的值为 : 13
a 的值为 : 14
a 的值为 : 15
```

`Break label` 语句：我们在 `for` 多层嵌套时，有时候需要直接跳出所有嵌套循环， 这时候就可以用到 go 的 label breaks 特征了。

先看一个范例代码：

```go
package main

import (
    "fmt"
)

func main() {
    fmt.Println("1")

Exit:
    for i := 0; i < 9; i++ {
        for j := 0; j < 9; j++ {
            if i+j > 15 {
                fmt.Print("exit")
                break Exit
            }
        }
    }

    fmt.Println("3")
}
```

执行效果：

```
1
exit3
```

注意：`label` 要写在 `for` 循环的开始而不是结束的地方。和 `goto` 是不一样的。虽然它是直接 `break` 退出到指定的位置。

`break` 的标签和 `goto` 的标签的区别可以参考下面代码：

```go
JLoop:
    for i := 0; i < 10; i++ {
        fmt.Println("label i is ", i)
        for j := 0; j < 10; j++ {
            if j > 5 {
                //跳到外面去啦，但是不会再进来这个for循环了
                break JLoop
            }
        }
    }

    //跳转语句 goto语句可以跳转到本函数内的某个标签
    gotoCount := 0
GotoLabel:
    gotoCount++
    if gotoCount < 10 {
        goto GotoLabel //如果小于10的话就跳转到GotoLabel
    }
```

`break` 标签除了可以跳出 `for` 循环，还可以跳出 `select` `switch` 循环， 参考下面代码：

```go
L:
    for ; count < 8192; count++ {
        select {
        case e := <-self.pIdCh:
            args[count] = e

        default:
            break L // 跳出 select 和 for 循环
        }

    }
```

## continue

`continue` 语句 跳过当前循环的剩余语句，然后继续进行下一轮循环。

Go 语言的 `continue` 语句 有点像 `break` 语句。但是 `continue` 不是跳出循环，而是跳过当前循环执行下一次循环语句。

`for` 循环中，执行 `continue` 语句会触发 `for` 增量语句的执行。

实例：

```go
package main

import "fmt"

func main() {
   /* 定义局部变量 */
   var a int = 10

   /* for 循环 */
   for a < 20 {
      if a == 15 {
         /* 跳过此次循环 */
         a = a + 1
         continue
      }
      fmt.Printf("a 的值为 : %d\n", a)
      a++    
   }  
}
```

以上实例执行结果为：

```
a 的值为 : 10
a 的值为 : 11
a 的值为 : 12
a 的值为 : 13
a 的值为 : 14
a 的值为 : 16
a 的值为 : 17
a 的值为 : 18
a 的值为 : 19
```

配合标签， `break` 和 `continue` 可在多级嵌套循环中跳出。

```go
func main() {
L1:
    for x := 0; x < 3; x++ {
L2:
        for y := 0; y < 5; y++ {
            if y > 2 { continue L2 }
            if x > 1 { break L1 }
            
            print(x, ":", y, " ")
        }
        println() 
    }
}
```

输出:

```
0:0  0:1  0:2
1:0  1:1  1:2
```

> **附: `break` 可用于 `for` 、 `switch` 、 `select` ，而 `continue` 仅能用于 `for` 循环。**

```go
x := 100

switch {
case x >= 0:
    if x == 0 { break }
    println(x)
}
```

goto、continue、break语句：

```go
package main

import "fmt"

func main() {

    //goto直接调到LAbEL2
    for {
        for i := 0; i < 10; i++ {
            if i > 3 {
                goto LAbEL2
            }
        }
    }
    fmt.Println("PreLAbEL2")
LAbEL2:
    fmt.Println("LastLAbEL2")

    //break跳出和LAbEL1同一级别的循环,继续执行其他的
LAbEL1:
    for {
        for i := 0; i < 10; i++ {
            if i > 3 {
                break LAbEL1
            }
        }
    }
    fmt.Println("OK")

    //continue
LABEL3:

    for i := 0; i < 3; i++ {
        for {
            continue LABEL3
        }
    }
    fmt.Println("ok")
}
```

输出如下：

```go
LastLAbEL2
OK
ok
```

# for与select嵌套退出循环解决方法

## break + 标签

使用golang中 `break` 的特性，在外层 `for` 加一个标签

```go
func test(){
	i := 0
	ForEnd:
	for {
		select {
		case <-time.After(time.Second * time.Duration(2)):
			i++
			if i == 5{
				fmt.Println("break now")
				break ForEnd
			}
			fmt.Println("inside the select: ")
		}
		fmt.Println("inside the for: ")
	}
}
```

## goto

使用 `goto` 直接跳出循环

```go
func test(){
	i := 0

	for {
		select {
		case <-time.After(time.Second * time.Duration(2)):
			i++
			if i == 5{
				fmt.Println("break now")
				goto ForEnd
			}
			fmt.Println("inside the select: ")
		}
		fmt.Println("inside the for: ")
	}
	ForEnd：
}
```

## 设置标记

```go
func test(){
    i := 0
    
    done := false

	for {
		select {
		case <-time.After(time.Second * time.Duration(2)):
			i++
			if i == 5{
				fmt.Println("break now")
				done = true
			}
        }
        
        if done {  // 跳出 for 
            break
        }
	}
}
```

## return

```go
func test(){
    i := 0
    

	for {
		select {
		case <-time.After(time.Second * time.Duration(2)):
			i++
			if i == 5{
				fmt.Println("break now")
				return
			}
        }
	}
}
```

# References

- [原文 - 6.循环控制Goto、Break、Continue](https://www.kancloud.cn/liupengjie/go/570042)
- [Go 语言中Select与for结合使用时可能会遇到的坑](https://studygolang.com/articles/5327)

---
title: Go语言的一个变量覆盖的问题
layout: post
categories: 填坑记录
---

原代码：

```go
var phone_conn net.TCPConn
// 从TCP 连接池中获取一个TCPConn，然后向服务端发送数据
for {
    phone_conn, err := phones[device_name].get_conn()
    if (net.TCPConn{}) == phone_conn || err != nil {
        log.Println(device_name, "test conn no phone conn error:", err)
        renderHtmlFileAndClose(conn, "net_error.html")
        return
    }

    data := []byte(testReq)
    _, err = phone_conn.Write(data)
    if err != nil {
        log.Println("send error", err)
    } else {
        break
    }
    // 发送失败的话，关闭连接重新选取一个TCPConn
    phone_conn.Close()
}

// 发送成功之后，则利用该TCPConn循环获取服务器端的返回
var buf = make([]byte, 4096)
for {
    n, err := phone_conn.Read(buf)

    log.Println(device_name, "test conn return ", n, ":", string(buf[:n]))

    if err == io.EOF {        
        break        
    }

    if err != nil {
        log.Println(device_name, "test conn read error:", err)
        break
    }
    // 打印返回的数据
    fmt.Println(string(buf[:n]))    
}
phone_conn.Close()
```

经过各种调整之后，我的程序中出现了上面的代码，结果TCP Conn（phone_conn） 始终read不出来数据，获取到n一直是0，以为是服务端出了问题，但是后来看到在服务端的日志中的的确确收到了请求并且返回了，然后日志中打印出来的错误是：

```
Invalid Argument
```

经过查阅资料，应该是文件句柄为空，在这里也就是这个TCPConn为空了，然后看到代码恍然大悟！局部变量同名变量重新定义之后和父级的同名变量是互不相关的，所以下面的代码在引用phone_conn的时候实际上引用的是没有赋值过的，所以把代码修改成下面的样子。


修改后的代码：

```go
var phone_conn net.TCPConn
var err error
for {
    phone_conn, err = phones[device_name].get_conn()
    if (net.TCPConn{}) == phone_conn || err != nil {
        log.Println(device_name, "test conn no phone conn error:", err)
        renderHtmlFileAndClose(conn, "net_error.html")
        return
    }

    data := []byte(testReq)
    _, err = phone_conn.Write(data)
    if err != nil {
        log.Println("send error", err)
    } else {
        break
    }
    phone_conn.Close()
}
```

总结一下，本来这个赋值语句是下面这样的，

```
phone_conn, err = phones[device_name].get_conn()
```

也就是没有问题的，后来前面的代码有调整，删掉了err的生命，原先err也是通过 := 这种语法声明的，然后这里的赋值就会有报错，我看见报错直接把=换成了:=，解决了编译错误，然后就造了这个bug，这里记下来，要小心go语言中的声明和定义语法以及局部变量覆盖的问题。
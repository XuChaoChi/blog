---
title: nsq源码阅读笔记--http模块(http_api)
tags:
  - nsq
  - go
category: 'nsq'
keywords: 'nsq,go'
date: 2019-10-17 23:32:05
---

# nsq源码阅读笔记--http模块(http_api)

### 代码地址

[nsq-http模块阅读](https://github.com/XuChaoChi/nsq_read/tree/master/internal/http_api)

### 简介

nsq中将标准库中的http模块又进行了简单的封装，目录在nsq项目中的 __internal/http_api__ 目录，封装了客户端和服务端，并对请求做了统一处理

### 目录组成

    ├── api_request.go          //客户端的封装
    ├── api_response.go         //请求处理的封装
    ├── compress.go             //协议回包压缩
    ├── http_server.go          //server部分的简单封装
    ├── req_params.go           //从url解析参数的简单封装
    └── topic_channel_args.go   //获取url中的topic和channel名字

<!--more-->

### api_response.go

在这个文件中主要实现对请求后的V1版本的回包装饰（包头、格式、日志记录）

- 使用到的结构体和方法定义

        //装饰器使用的函数
        //将原有的APIHandler外层装饰一层其他逻辑返回装饰后的APIHandler
        type Decorator func(APIHandler) APIHandler

        //api逻辑处理函数的定义（httprouter 的回调格式）
        type APIHandler func(http.ResponseWriter, *http.Request, httprouter.Params) (interface{}, error)

        //记录错误的结构体
        type Err struct {
            Code int
            Text string
        }


- 使用装饰者模式，将 __httprouter.Handle__ 的逻辑装饰一些统一逻辑:回包(V1)，日志(Log)，输出(PlainText)）,以 __http_api.Decorate(testHandler, http_api.V1, log)__ 为例:最后的调用为log->http_api.V1->testHandler

        func Decorate(f APIHandler, ds ...Decorator) httprouter.Handle {
            decorated := f
            for _, decorate := range ds {
                //每次遍历都是将原来的APIHandler外面再装饰一层，最后的调用顺序FILO
                decorated = decorate(decorated)
            }
            return func(w http.ResponseWriter, req *http.Request, ps httprouter.Params) {
                //调用最后封装的结果
                decorated(w, req, ps)
            }
        }


- 使用 __github.com/julienschmidt/httprouter__ 来作为路由模块 
    - 使用例子
  
            router.Handle("GET", "/test", http_api.Decorate(pingHandler, http_api.V1, log))

    - 同时nsq对httprouter.Router中的异常处理进行了简单封装

            //最后返回一个包装后的APIHandler
            func LogPanicHandler(logf lg.AppLogFunc) func(w http.ResponseWriter, req *http.Request, p interface{}) {
                return func(w http.ResponseWriter, req *http.Request, p interface{}) {
                    logf(lg.ERROR, "panic in HTTP handler - %s", p)
                    Decorate(func(w http.ResponseWriter, req *http.Request, ps httprouter.Params) (interface{}, error) {
                        return nil, Err{500, "INTERNAL_ERROR"}
                    }, Log(logf), V1)(w, req, nil)
                }
            }

            //httprouter中使用
            router.PanicHandler = http_api.LogPanicHandler(logf)
            router.NotFound = http_api.LogNotFoundHandler(logf)
            router.MethodNotAllowed = http_api.LogMethodNotAllowedHandler(logf)


### api_request.go

这个文件中主要实现了一个具有连接超时和请求超时的http客户端，以及V1版本的POST和GET的简单封装

- 客户端的封装

        //Http客户端支持超时（连接超时，请求超时）
        func NewDeadlineTransport(connectTimeout time.Duration, requestTimeout time.Duration) *http.Transport {
            // arbitrary values copied from http.DefaultTransport
            transport := &http.Transport{
                DialContext: (&net.Dialer{
                    Timeout:   connectTimeout,
                    KeepAlive: 30 * time.Second,
                    DualStack: true,
                }).DialContext,
                ResponseHeaderTimeout: requestTimeout,
                MaxIdleConns:          100,
                IdleConnTimeout:       90 * time.Second,
                TLSHandshakeTimeout:   10 * time.Second,
            }
            return transport
        }

        //包装的http客户端结构
        type Client struct {
            c *http.Client
        }

        //创建一个新的客户端
        func NewClient(tlsConfig *tls.Config, connectTimeout time.Duration, requestTimeout time.Duration) *Client {
            transport := NewDeadlineTransport(connectTimeout, requestTimeout)
            transport.TLSClientConfig = tlsConfig
            return &Client{
                c: &http.Client{
                    Transport: transport,
                    Timeout:   requestTimeout,
                },
            }
        }

        // GETV1 is a helper function to perform a V1 HTTP request
        // and parse our NSQ daemon's expected response format, with deadlines.
        //GETV1是V1版本的http get方法
        //将get的内容解析成json
        func (c *Client) GETV1(endpoint string, v interface{}) error {
        retry:
            //新建一个Get请求
            req, err := http.NewRequest("GET", endpoint, nil)
            if err != nil {
                return err
            }

            req.Header.Add("Accept", "application/vnd.nsq; version=1.0")
            resp, err := c.c.Do(req)
            if err != nil {
                return err
            }
            //获取返回协请求body中的内容
            body, err := ioutil.ReadAll(resp.Body)
            resp.Body.Close()
            if err != nil {
                return err
            }
            //判断请求状态是否是成功
            if resp.StatusCode != 200 {
                //如果是403尝试下将http地转换成https尝试下
                if resp.StatusCode == 403 && !strings.HasPrefix(endpoint, "https") {
                    endpoint, err = httpsEndpoint(endpoint, body)
                    if err != nil {
                        return err
                    }
                    goto retry
                }
                return fmt.Errorf("got response %s %q", resp.Status, body)
            }
            //将body中的内容转换为v
            err = json.Unmarshal(body, &v)
            if err != nil {
                return err
            }

            return nil
        }

        // PostV1 is a helper function to perform a V1 HTTP request
        // and parse our NSQ daemon's expected response format, with deadlines.
        //GETV1是V1版本的http post方法
        //将get的内容解析成json
        func (c *Client) POSTV1(endpoint string) error {
        retry:
            req, err := http.NewRequest("POST", endpoint, nil)
            if err != nil {
                return err
            }

            req.Header.Add("Accept", "application/vnd.nsq; version=1.0")

            resp, err := c.c.Do(req)
            if err != nil {
                return err
            }

            body, err := ioutil.ReadAll(resp.Body)
            resp.Body.Close()
            if err != nil {
                return err
            }
            if resp.StatusCode != 200 {
                //如果是403尝试下将http地转换成https尝试下
                if resp.StatusCode == 403 && !strings.HasPrefix(endpoint, "https") {
                    endpoint, err = httpsEndpoint(endpoint, body)
                    if err != nil {
                        return err
                    }
                    goto retry
                }
                return fmt.Errorf("got response %s %q", resp.Status, body)
            }

            return nil
        }

        //将请求地址转换为https
        func httpsEndpoint(endpoint string, body []byte) (string, error) {
            var forbiddenResp struct {
                HTTPSPort int `json:"https_port"`
            }
            //body中的端口
            err := json.Unmarshal(body, &forbiddenResp)
            if err != nil {
                return "", err
            }
            //获取url
            u, err := url.Parse(endpoint)
            if err != nil {
                return "", err
            }
            //获取url中的主机地址
            host, _, err := net.SplitHostPort(u.Host)
            if err != nil {
                return "", err
            }

            u.Scheme = "https"
            u.Host = net.JoinHostPort(host, strconv.Itoa(forbiddenResp.HTTPSPort))
            return u.String(), nil
        }

### http_server.go

这个文件针对http的server进行统一简单的封装（日志和服务的监听）

    //Param1监听对象
    //Param2路由
    //Param3协议类型
    //Param4日志函数
    func Serve(listener net.Listener, handler http.Handler, proto string, logf lg.AppLogFunc) error {
        logf(lg.INFO, "%s: listening on %s", proto, listener.Addr())
        //创建HTTP对象
        server := &http.Server{
            Handler: handler,
            //错误日志
            ErrorLog: log.New(logWriter{logf}, "", 0),
        }
        err := server.Serve(listener)
        // 排除正常关闭的情况
        // theres no direct way to detect this error because it is not exposed
        if err != nil && !strings.Contains(err.Error(), "use of closed network connection") {
            return fmt.Errorf("http.Serve() error - %s", err)
        }

        logf(lg.INFO, "%s: closing %s", proto, listener.Addr())

        return nil
    }

### topic_channel_args.go

- 定义了url参数获取的接口
    
        type getter interface {
            Get(key string) (string, error)
        }

- 以及一个使用getter接口作为参数的函数，主要通过url参数来获取topic和channel的名字

        func GetTopicChannelArgs(rp getter) (string, string, error)

### req_params.go

实现的getter接口,从http.Request来获取url中的参数

- getter接口结构体
  
        type ReqParams struct {
            url.Values
            Body []byte
        }

- 从http.Request初始化
 
        func NewReqParams(req *http.Request) (*ReqParams, error) {
            reqParams, err := url.ParseQuery(req.URL.RawQuery)
            if err != nil {
                return nil, err
            }

            data, err := ioutil.ReadAll(req.Body)
            if err != nil {
                return nil, err
            }

            return &ReqParams{reqParams, data}, nil
        }

- Get方法参数的获取

        func (r *ReqParams) Get(key string) (string, error) {
            v, ok := r.Values[key]
            if !ok {
                return "", errors.New("key not in query params")
            }
            return v[0], nil
        }

### compress.go

如果客户端请求支持Accept-Encoding gzip 则压缩responses

    func CompressHandler(h http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        L:
            for _, enc := range strings.Split(r.Header.Get("Accept-Encoding"), ",") {
                switch strings.TrimSpace(enc) {
                case "gzip":
                    w.Header().Set("Content-Encoding", "gzip")
                    w.Header().Add("Vary", "Accept-Encoding")

                    gw := gzip.NewWriter(w)
                    defer gw.Close()
                    //使用hijack接管http
                    h, hok := w.(http.Hijacker)
                    if !hok { /* w is not Hijacker... oh well... */
                        h = nil
                    }

                    w = &compressResponseWriter{
                        Writer:         gw,
                        ResponseWriter: w,
                        Hijacker:       h,
                    }

                    break L
                case "deflate":
                    w.Header().Set("Content-Encoding", "deflate")
                    w.Header().Add("Vary", "Accept-Encoding")

                    fw, _ := flate.NewWriter(w, flate.DefaultCompression)
                    defer fw.Close()

                    h, hok := w.(http.Hijacker)
                    if !hok { /* w is not Hijacker... oh well... */
                        h = nil
                    }

                    w = &compressResponseWriter{
                        Writer:         fw,
                        ResponseWriter: w,
                        Hijacker:       h,
                    }

                    break L
                }
            }
            //进行路由逻辑
            h.ServeHTTP(w, r)
        })
    }

### 使用的例子

#### server例子

    package main

    import (
        "fmt"
        "log"
        "net"
        "net/http"
        "os"

        "github.com/julienschmidt/httprouter"
        "github.com/nsqio/nsq/internal/http_api"
        "github.com/nsqio/nsq/internal/lg"
    )

    type TestLogger lg.Logger

    var testLog TestLogger

    func logf(level lg.LogLevel, f string, args ...interface{}) {
        lg.Logf(testLog, lg.DEBUG, level, f, args...)
    }

    func main() {
        testLog = log.New(os.Stderr, "[test]", log.Ldate|log.Ltime|log.Lmicroseconds)
        logf(lg.DEBUG, "test")
        log := http_api.Log(logf)

        router := httprouter.New()
        router.HandleMethodNotAllowed = true
        router.PanicHandler = http_api.LogPanicHandler(logf)
        router.NotFound = http_api.LogNotFoundHandler(logf)
        router.MethodNotAllowed = http_api.LogMethodNotAllowedHandler(logf)

        router.Handle("GET", "/test", http_api.Decorate(testHandler, http_api.V1, log))
        //router.Handle("GET", "/test", http_api.Decorate(testHandler, log, http_api.V1))

        listenter, err := net.Listen("tcp", "0.0.0.0:8080")
        if err != nil {
            fmt.Println(err)
        }

        http_api.Serve(listenter, http_api.CompressHandler(router), "test", logf)
    }

    type JSONStruct struct {
        Msg string `json:"msg"`
    }

    func testHandler(w http.ResponseWriter, req *http.Request, ps httprouter.Params) (interface{}, error) {
        r, _ := http_api.NewReqParams(req)
        topic, channel, _ := http_api.GetTopicChannelArgs(r)
        logf(lg.DEBUG, "topic:%s, channel:%s", topic, channel)
        data := JSONStruct{Msg: fmt.Sprintf("topic:%s, channel:%s", topic, channel)}
        return data, nil
    }


#### client例子

    package main

    import (
        "fmt"
        "log"
        "net/http"
        "os"
        "time"

        "github.com/nsqio/nsq/internal/http_api"
        "github.com/nsqio/nsq/internal/lg"
    )

    type MyHandler uint32

    func (self MyHandler) ServeHTTP(w http.ResponseWriter, req *http.Request) {
        w.WriteHeader(http.StatusNotFound)
        fmt.Fprintf(w, "no such page: %s\n", req.URL)
    }

    type TestLogger lg.Logger

    var testLog TestLogger

    func logf(level lg.LogLevel, f string, args ...interface{}) {
        lg.Logf(testLog, lg.DEBUG, level, f, args...)
    }

    func main() {
        testCli()
    }

    type JSONStruct struct {
        Msg string `json:"msg"`
    }

    func testCli() {
        testLog = log.New(os.Stderr, "[test]", log.Ldate|log.Ltime|log.Lmicroseconds)
        c := http_api.NewClient(nil, 1*time.Second, 1*time.Second)
        data := JSONStruct{}
        err := c.GETV1("http://127.0.0.1:8080/test?topic=ttt&channel=cccc", &data)
        if err != nil {
            logf(lg.ERROR, "%s", err)
        } else {
            logf(lg.DEBUG, "data:%s", data.Msg)
        }
    }


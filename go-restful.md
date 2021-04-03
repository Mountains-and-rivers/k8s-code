### demo-01

```
mkdir test_project

cd test_project

go mod init

export GOPROXY="https://goproxy.io"

export GO111MODULE=on

go mod download
修改main.go
package main
import(
    "fmt"
    "github.com/emicklei/go-restful"
    "net/http"
)

func main() {
    fmt.Printf("runing.....\n")
    ws := new(restful.WebService)
    ws.Path("/hello")
    ws.Route(ws.GET("/").To(ping).Consumes(restful.MIME_JSON).Produces(restful.MIME_JSON))
    container := restful.NewContainer()
    container.Add(ws)

   http.ListenAndServe(":8080", container)

}
type Resp struct{
    Code int
    Msg string
    //data interface{}
}

func ping(request *restful.Request, response *restful.Response) {
    fmt.Printf("pong.....\n")
    data := Resp{Code:0, Msg:"test"}
    response.WriteEntity(data)
}

go mod tidy #add missing and remove unused modules

go run main.go

访问localhost:8080/hello
```

### demo-02

```
package main

import (
	"github.com/emicklei/go-restful"
	"log"
	"net/http"
	"os"
)

func main() {
	wsContainer := restful.NewContainer()
	wsContainer.Router(restful.CurlyRouter{})

	ws := new(restful.WebService)
	ws.
		Path("/api").
		Consumes(restful.MIME_JSON).
		Produces(restful.MIME_JSON) // 可单独设置每一个方法

	//	最终地址 https://127.0.0.1:7443/api/config
	ws.Route(ws.POST("/config").Filter(basicAuthenticate).To(config))

	ws.Filter(basicAuthenticate)//basic auth filter

	wsContainer.Add(ws)
	log.Printf("start listening on localhost:7443")
	server := &http.Server{Addr: ":7443", Handler: wsContainer}

	//证书生成方式参考https://blog.csdn.net/u011411069/article/details/79994716
	crtPath := "/home/leen/Desktop/certificate.crt"//crt 证书
	_, err := os.Stat(crtPath)
	if err != nil {
		log.Println("crt file not exist!")
		return
	}

	keyPath := "/home/leen/Desktop/certificate.key"//key 证书
	_, err = os.Stat(crtPath)
	if err != nil {
		log.Println("key file not exist!")
		return
	}

	err = server.ListenAndServeTLS(crtPath, keyPath)
	if err != nil {
		log.Println(err.Error())
	}
}

//basic auth 验证过滤
func basicAuthenticate(req *restful.Request, resp *restful.Response, chain *restful.FilterChain) {
	// usr/pwd = admin/admin
	u, p, ok := req.Request.BasicAuth()
	if !ok || u != "admin" || p != "admin" {
		resp.AddHeader("WWW-Authenticate", "Basic realm=Protected Area")
		resp.WriteErrorString(401, "401: Not Authorized")
		return
	}
	chain.ProcessFilter(req, resp)
}

//rest请求处理方法
func config(request *restful.Request, response *restful.Response) {
	entity := new(model.TestEntity)
	err := request.ReadEntity(entity)

	if err != nil {
		response.WriteEntity(model.NewResult(500, err.Error()))
	} else {
		response.WriteEntity(model.NewResult(0, "config successful!"))
	}
}
```


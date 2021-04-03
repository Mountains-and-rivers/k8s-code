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


#http.Request重用

> 针对除了Get以外的请求

```go
package main

import (
	"net/http"
	"strings"
	)

func main(){
	    reader := strings.NewReader("hello")
    	req,_ := http.NewRequest("POST","http://www.abc.com",reader)
    	client := http.Client{}
    	client.Do(req) // 第一次会请求成功
    	client.Do(req) // 请求失败
}
```

第二次请求会出错
```
http: ContentLength=5 with Body length 0
```
原因是第一次请求后req.Body已经读取到结束位置，所以第二次请求时无法再读取body，
解决方法：重写req.Body的方法

```go
package reader

import (
	"io"
	"net/http"
	"strings"
	"sync/atomic"
)

type Repeat struct{
	reader io.ReaderAt
	offset int64
}

func (p *Repeat) Read(val []byte) (n int, err error) {
	n, err = p.reader.ReadAt(val, p.offset)
	atomic.AddInt64(&p.offset, int64(n))
	return
}

func (p *Repeat) Close() error {
    // 因为req.Body实现了readcloser接口，所以要实现close方法
    // 但是repeat中的值有可能是只读的，所以这里只是尝试关闭一下。
	if p.reader != nil {
    		if rc, ok := p.reader.(io.Closer); ok {
    			return rc.Close()
    		}
    	}
	return nil
}

func doPost()  {
    client := http.Client{}
    reader := strings.NewReader("hello")
    req , _ := http.NewRequest("POST","http://www.abc.com",reader)
    req.Body = &Repeat{reader:reader,offset:0}
    client.Do(req)
    req.Body = &Repeat{reader:reader,offset:0}
    client.Do(req)
}
```

这样就不会报错了，因为也重写了Close()方法，所以同时也解决了request重用时，req.Body自动关闭的问题。
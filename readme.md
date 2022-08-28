# Sinx使用说明

快速开始

1.client

```go
package main

import (
	"fmt"
	"io"
	"net"
	"sinx/simodel"
	"time"
)

func main() {
	addr, _ := net.ResolveTCPAddr("tcp", "127.0.0.1:33366")
	conn, err := net.DialTCP("tcp", nil, addr)
	if err != nil {
		fmt.Println("client", err)
		return
	}
	var sendid uint32 = 101
    // 开启监听协程，让发送和接收不干扰
	go ListenConn(conn)
	for {
		// 创建信息打包，封装模块，并将消息封装。
		pack := simodel.NewPackMsgModel()
		msg, err := pack.PackMsgByOther(sendid, []byte("client向server发送了条消息，id为101号。finish。"))
		if err != nil {
			fmt.Println("client have error", err)
			continue
		}
        
        // 发送消息
		_, err = conn.Write(msg)
		if err != nil && err == io.EOF {
			fmt.Printf("server断开了conn（可能原因：server已到设定最大连接上线）")
			return
		} else if err != nil {
			fmt.Println("server断开了conn（可能原因：server已到设定最大连接上线）", err)
			return
		}
		time.Sleep(time.Second * 1)
	}
}

func ListenConn(conn *net.TCPConn) {
    // 创建信息打包，封装模块。
	pack := simodel.NewPackMsgModel()
	for {
		// 创建容器并接收来自服务器的消息。
		buf := make([]byte, 512)
		n, err := conn.Read(buf)
		if err != nil && err == io.EOF {
			fmt.Printf("server断开了conn（可能原因：server已到设定最大连接上线）")
			return
		} else if err != nil {
			fmt.Println("server断开了conn（可能原因：server已到设定最大连接上线）", err)
			return
		}
        
        // 解包来自服务器的消息
		msg, err := pack.UnPackMsg(buf[:n])
		if err != nil {
			fmt.Println("client unpack 出现了错误", err)
			return
		}
		fmt.Printf("client接收到id为:%d-的新消息，内容为%s\n", msg.GetMsgID(), msg.GetData())
	}
}

```

2.Server

```go
package main

import (
	"fmt"
	"sinx/siface"
	"sinx/simodel"
	"strconv"
)

# 1.创建服务器
SinxServer := simodel.NewSinxServer()

# 2.定义路由处理业务逻辑
type msg101 struct {
	simodel.RouteModel
}

func (s *msg101) Handle(req siface.RequestFace) {
	recmsg := req.GetMsg()
	fmt.Printf("msg101 : server收到%d号消息:%s\n", recmsg.GetMsgID(), string(recmsg.GetData()))
	req.GetMsg()
	pack := simodel.NewPackMsgModel()
	msg, _ := pack.PackMsgByOther(99, []byte("s101_server返回消息id99。"))
	_, err := req.GetConn().GetTCPConn().Write(msg)
	if err != nil {
		fmt.Println("server send error", err)
	}
	return
}


# 3.添加处理路由
SinxServer.AddRouth(101,&msg101{})

# 4.启动服务器
SinServer.Serve()
```


## golang web-server design

### server graceful shutdown and double-check forced shutdown

```golang
package server

import (
	"context"
	"github.com/gin-gonic/gin"
	"go-snippets/internal/pkg/log"
	"net/http"
	"os"
	"os/signal"
	"sync"
	"syscall"
	"time"
)

type Server struct {
	Router *gin.Engine
	Addr   string
}

func New(router *gin.Engine, addr string) *Server {
	return &Server{
		Router: router,
		Addr:   addr,
	}
}


func (s *Server) httpServer(stop <-chan struct{}, group *sync.WaitGroup) {

	defer 	group.Done()

	srv := &http.Server{
		Addr:    s.Addr,
		Handler: s.Router,
	}

	// Initializing the server in a goroutine so that
	// it won't block the graceful shutdown handling below
	// http server
	go func() {
		log.Infof("server listen on %v", srv.Addr)
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Fatalf("listen: %s\n", err)
		}
	}()

	<- stop
	// Wait for interrupt signal to gracefully shutdown the server with
	// a timeout of 5 seconds.
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	if err := srv.Shutdown(ctx);err != nil {
		log.Errorf("server shutdown failed: %v",err.Error())
	} else {
		time.Sleep(100 * time.Second)
		log.Info("http server shutdown successfully")
	}

}

func (s *Server) ServeAndListen() error {

	// 统一关机信号，用于通知所有server停止服务，例如http、grpc等
	stop := make(chan struct{})

	// 用于等待所有服务优雅关机
	group:= sync.WaitGroup{}

	// 每启动一个server，就添加1
	group.Add(1)
	go s.httpServer(stop, &group)

	// 系统关机信号，由外部触发，例如ctrl+c等
	shutdownHandler := make(chan os.Signal, 2)

	// 补货所有的打断信后
	// kill -2 is syscall.SIGINT
	// kill -9 is syscall.SIGKILL but can't be catch, so don't need add it
	signal.Notify(shutdownHandler, syscall.SIGINT, syscall.SIGTERM)

	go func() {
		// 第一次关机信号，优雅的关机
		<-shutdownHandler
		log.Info("Shutting down server...")
		// The context is used to inform the server it has 5 seconds to finish
		// the request it is currently handling
		close(stop)

		// 第二次关机信号，强制退出
		<- shutdownHandler
		log.Warn("Forced shutdown server...")

		os.Exit(1) // second signal. Exit directly. }()
	}()

	group.Wait()

	return nil
}s
```

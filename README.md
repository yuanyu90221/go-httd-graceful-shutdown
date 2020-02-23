# go graceful shutdown machinism

# reference
[go graceful shutdown](https://www.youtube.com/watch?v=nCn1Wmi6Wug)

## question description

    when server suddenly hit situation, server connect shutdown directly not graceful for connected proccess

    go httd module provide a Shutdown method to Gracefully handle connected process before shutdown

## prerequest

    go version must great than 1.8

## sample to handle this
gin server with restful api
```golang===
func main() {
    router := gin.Default()
	router.GET("/", func(c *gin.Context) {
		time.Sleep(5 * time.Second)
		c.String(http.StatusOK, "Welcome Gin Server\n")
	})

	srv := &http.Server{
		Addr:    ":8080",
		Handler: router,
	}
	
    if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
        log.Fatalf("listen: %s\n", err)
    }
}
```
## solution

    1 move listen handler into goroutine(make server handler process into daemon)
    2 setup a channel to listen SIGINT and SIGTERM for shutdown signal
    3 setup withTimeout for preserve how many time to close the connection

```golang===
func main() {
    router := gin.Default()
	router.GET("/", func(c *gin.Context) {
		time.Sleep(5 * time.Second)
		c.String(http.StatusOK, "Welcome Gin Server\n")
	})

	srv := &http.Server{
		Addr:    ":8080",
		Handler: router,
	}
    //1 set up handler to go routine
	go func() {
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Fatalf("listen: %s\n", err)
		}
	}()
    // 2 make channel for listen os.Siganal and setup notify Signal
	quit := make(chan os.Signal, 1)

	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit
	log.Println("Shutdown Server ...")

    // 3 setup withTimeout to preserve connection before close
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	if err := srv.Shutdown(ctx); err != nil {
		log.Fatal("Server Shutdown: ", err)
	}
	log.Println("Server exiting")
}
```
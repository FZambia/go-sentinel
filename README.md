go-sentinel
===========

Redis Sentinel support for [redigo](https://github.com/garyburd/redigo) library.

**API is unstable and can change at any moment** – use with tools like Glide, Godep etc.

Documentation
-------------

- [API Reference](http://godoc.org/github.com/FZambia/go-sentinel)

Quick Start
-----------

```golang
package main

import (
	"fmt"
	"log"

	"github.com/garyburd/redigo/redis"
	"github.com/FZambia/go-sentinel"
)

var redisClient *redis.Pool


// Sentinel factory
func sentinelPool() *redis.Pool {
 	sntnl := &sentinel.Sentinel{
 		// format host:port or just :port
 		Addrs:      []string{":26379", ":26380", ":26381"},
 		MasterName: "mymaster",
 		Dial: func(addr string) (redis.Conn, error) {
 			timeout := 500 * time.Millisecond
 			c, err := redis.DialTimeout("tcp", addr, timeout, timeout, timeout)
 			if err != nil {
 				return nil, err
 			}
 			return c, nil
 		},
 	}
 	return &redis.Pool{
 		MaxIdle:     3,
 		MaxActive:   64,
 		Wait:        true,
 		IdleTimeout: 240 * time.Second,
 		Dial: func() (redis.Conn, error) {
 			// get relevant address for master
 			masterAddr, err := sntnl.MasterAddr()
 			if err != nil {
 				return nil, err
 			}
 			c, err := redis.Dial("tcp", masterAddr)
 			if err != nil {
 				return nil, err
 			}
 			return c, nil
 		},
 		TestOnBorrow: func(c redis.Conn, t time.Time) error {
 			if !sentinel.TestRole(c, "master") {
 				return errors.New("Role check failed")
 			} else {
 				return nil
 			}
 		},
 	}
 }

func main() {
	redisClient = sentinelPool()
	c, err := redisClient.Dial()

	if err != nil {
		log.Fatalf("ERROR: %s", err)
	}

	// do something with redis
	c.Do("SET", "foo", "bar")
}

```

Alternative solution
--------------------

You can alternatively configure Haproxy between your application and Redis to proxy requests to Redis master instance if you only need HA:

```
listen redis
    server redis-01 127.0.0.1:6380 check port 6380 check inter 2s weight 1 inter 2s downinter 5s rise 10 fall 2
    server redis-02 127.0.0.1:6381 check port 6381 check inter 2s weight 1 inter 2s downinter 5s rise 10 fall 2 backup
    bind *:6379
    mode tcp
    option tcpka
    option tcplog
    option tcp-check
    tcp-check send PING\r\n
    tcp-check expect string +PONG
    tcp-check send info\ replication\r\n
    tcp-check expect string role:master
    tcp-check send QUIT\r\n
    tcp-check expect string +OK
    balance roundrobin
```

This way you don't need to use this library.

License
-------

Library is available under the [Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0.html). 

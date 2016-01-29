#LARS HTTP Router

![Project status](http://img.shields.io/status/experimental.png?color=red)

lars (Library Access/Retrieval System), is a fast radix-tree based, zero allocation, HTTP router for Go.


![test gif](examples/README/test.gif)


Why Another HTTP Router?
------------------------
I have noticed that most routers out there, IMHO, are adding too much functionality that doesn't belong in an HTTP router, and they are turning into web frameworks, with all the bloat that entails. LARS aims to remain a simple yet powerful HTTP router that can be plugged into any existing framework; furthermore LARS allowing the passing of global + application variables that comply with it's IGlobals interface (right on the Context object) makes frameworks redundant as **LARS wraps the framework instead of the framework wrapping LARS**.<add link to an example here>

Unique Features 
--------------
* Context allows the passing of framework/globals/application specific variables via it's Globals field.
  * The Globals object is essentially all of the application specific variables and libraries needed by your handlers and functions, keeping a clear separation between your http and application contexts.
* Handles mutiple url patterns not supported by many other routers.
* Contains helpful logic to help prevent adding bad routes, keeping your url's consistent.
  * i.e. /user/:id and /user/:user_id - the second one will fail to add letting you know that :user_id should be :id
* Has an uber simple middleware + handler definitions!!! middleware and handlers actually have the exact same definition!



Installation
-----------

Use go get 

```go
go get github.com/go-playground/lars
``` 

or to update

```go
go get -u github.com/go-playground/lars
``` 

Then import lars package into your code.

```go
import "github.com/go-playground/lars"
``` 

Usage
------
```go
package main

import (
	"log"
	"net/http"
	"os"
	"time"

	"github.com/go-playground/lars"
)

// This is a contrived example using globals as I would use it in production
// I would break things into separate files but all here for simplicity

// ApplicationGlobals houses all the application info for use.
type ApplicationGlobals struct {
	// DB - some database connection
	Log *log.Logger
	// Translator - some i18n translator
	// JSON - encoder/decoder
	// Schema - gorilla schema
	// .......
}

// Reset gets called just before a new HTTP request starts calling
// middleware + handlers
func (g *ApplicationGlobals) Reset(c *lars.Context) {
	// DB = new database connection or reset....
	//
	// We don't touch translator + log as they don't change per request
}

// Done gets called after the HTTP request has completed right before
// Context gets put back into the pool
func (g *ApplicationGlobals) Done() {
	// DB.Close()
}

var _ lars.IGlobals = &ApplicationGlobals{} // ensures ApplicationGlobals complies with lasr.IGlobals at compile time

func main() {

	logger := log.New(os.Stdout, "INFO: ", log.Ldate|log.Ltime|log.Lshortfile)
	// translator := ...
	// db := ... base db connection or info
	// json := ...
	// schema := ...

	globalsFn := func() lars.IGlobals {
		return &ApplicationGlobals{
			Log: logger,
			// Translator: translator,
			// DB: db,
			// JSON: json,
			// schema:schema,
		}
	}

	l := lars.New()
	l.RegisterGlobals(globalsFn)
	l.Use(Logger)

	l.Get("/", Home)

	users := l.Group("/users")
	users.Get("", Users)

	// you can break it up however you with, just demonstrating that you can
	// have groups of group
	user := users.Group("/:id")
	user.Get("", User)
	user.Get("/profile", UserProfile)

	http.ListenAndServe(":3007", l.Serve())
}

// Home ...
func Home(c *lars.Context) {

	app := c.Globals.(*ApplicationGlobals)

	var username string

	// username = app.DB.find(user by .....)

	app.Log.Println("Found User")

	c.Response().Write([]byte("Welcome Home " + username))
}

// Users ...
func Users(c *lars.Context) {

	app := c.Globals.(*ApplicationGlobals)

	app.Log.Println("In Users Function")

	c.Response().Write([]byte("Users"))
}

// User ...
func User(c *lars.Context) {

	app := c.Globals.(*ApplicationGlobals)

	id, _ := c.Param("id")

	var username string

	// username = app.DB.find(user by id.....)

	app.Log.Println("Found User")

	c.Response().Write([]byte("Welcome " + username + " with id " + id))
}

// UserProfile ...
func UserProfile(c *lars.Context) {

	app := c.Globals.(*ApplicationGlobals)

	id, _ := c.Param("id")

	var profile string

	// profile = app.DB.find(user profile by .....)

	app.Log.Println("Found User Profile")

	c.Response().Write([]byte("Here's your profile " + profile + " user " + id))
}

// Logger ...
func Logger(c *lars.Context) {

	req := c.Request()

	start := time.Now()

	c.Next()

	stop := time.Now()
	path := req.URL.Path

	if path == "" {
		path = "/"
	}

	log.Printf("%s %d %s %s", req.Method, c.Response().Status(), path, stop.Sub(start))
}

```

Benchmarks
-----------
Run on MacBook Pro (Retina, 15-inch, Late 2013) 2.6 GHz Intel Core i7 16 GB 1600 MHz DDR3 using Go version go1.5.3 darwin/amd64


```go
go test -bench=. -benchmem=true
#GithubAPI Routes: 203
   LARS: 84792 Bytes

#GPlusAPI Routes: 13
   LARS: 7240 Bytes

#ParseAPI Routes: 26
   LARS: 8160 Bytes

#Static Routes: 157
   LARS: 82168 Bytes

PASS
BenchmarkLARS_Param       	20000000	        88.6 ns/op	       0 B/op	       0 allocs/op
BenchmarkLARS_Param5      	10000000	       145 ns/op	       0 B/op	       0 allocs/op
BenchmarkLARS_Param20     	 5000000	       376 ns/op	       0 B/op	       0 allocs/op
BenchmarkLARS_ParamWrite  	10000000	       163 ns/op	       0 B/op	       0 allocs/op
BenchmarkLARS_GithubStatic	20000000	       110 ns/op	       0 B/op	       0 allocs/op
BenchmarkLARS_GithubParam 	10000000	       156 ns/op	       0 B/op	       0 allocs/op
BenchmarkLARS_GithubAll   	   50000	     38378 ns/op	       0 B/op	       0 allocs/op
BenchmarkLARS_GPlusStatic 	20000000	        70.5 ns/op	       0 B/op	       0 allocs/op
BenchmarkLARS_GPlusParam  	20000000	       100 ns/op	       0 B/op	       0 allocs/op
BenchmarkLARS_GPlus2Params	10000000	       142 ns/op	       0 B/op	       0 allocs/op
BenchmarkLARS_GPlusAll    	 1000000	      1894 ns/op	       0 B/op	       0 allocs/op
BenchmarkLARS_ParseStatic 	20000000	        97.0 ns/op	       0 B/op	       0 allocs/op
BenchmarkLARS_ParseParam  	20000000	       115 ns/op	       0 B/op	       0 allocs/op
BenchmarkLARS_Parse2Params	10000000	       130 ns/op	       0 B/op	       0 allocs/op
BenchmarkLARS_ParseAll    	  500000	      3890 ns/op	       0 B/op	       0 allocs/op
BenchmarkLARS_StaticAll   	   50000	     23509 ns/op	       0 B/op	       0 allocs/op

```

This package is inspired by the following 
- [httptreemux](https://github.com/dimfeld/httptreemux)
- [httprouter](https://github.com/julienschmidt/httprouter)
- [echo](https://github.com/labstack/echo)

License 
--------
This project is licensed unter MIT, for more information look into the LICENSE file.
Copyright (c) 2016 Go Playground



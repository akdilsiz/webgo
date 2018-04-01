[![Build Status](https://travis-ci.org/bnkamalesh/webgo.svg?branch=master)](https://travis-ci.org/bnkamalesh/webgo)
[![](https://goreportcard.com/badge/github.com/bnkamalesh/webgo)](https://goreportcard.com/report/github.com/bnkamalesh/webgo)

# WebGo v2.0

WebGo is a minimalistic framework for Go. It primarily gives you the following abilities.

1. Multiplexer
2. Chaining handlers
3. Middlewares
4. Webgo context
5. Helper functions

WebGo's route handlers have the same signature as the standard libraries HTTP handler.
i.e. `http.HandlerFunc`

### Multiplexer

WebGo multiplexer lets you define URIs with or without parameters. Following are the ways you can 
define a URI for a route

1. `api/users` - no URI parameters
2. `api/users/:userID` - named URI parameter; the value would be available in the parameter `userID`
3. `api/:wildcard*` - wildcard URIs; every router confirming to `/api/path/to/anything` would be
matched by this route, and the path would be available inside the named URI parameter `wildcard`

### Chaining

Chaining lets you hanlde a route with multiple handlers, and they will be executed in sequence.
All handlers in the chain are `http.HandlerFunc`s.

### Middlewares

Webgo's middleware signature is `func(http.ResponseWriter, *http.Request, http.HandlerFunc)`.
Its `router` exposes a method `Use` to add middlewares to the app. e.g.

```
func accessLog(rw http.ResponseWriter, req *http.Request, next http.HandlerFunc) {
	start := time.Now()
	next(rw, req)
	end := time.Now()
	log := end.Format("2006-01-02 15:04:05 -0700 MST") + " " + req.Method + " " + req.URL.String() + " " + end.Sub(start).String()
	fmt.Println(log)
}

router := webgo.NewRouter(*webgo.Config, []*Route)
router.Use(accessLog)
```

### Context

Any app specific context can be be set inside the router, and is made available inside every request
handler via HTTP request context. The webgo Context also provides the route configuration itself 
as well as the URI parameters.

```
router := webgo.NewRouter(*webgo.Config, []*Route)
router.AppContext = map[string]interface{}{
	"dbHandler": <db handler>
}

func API1(rw http.ResponseWriter, req *http.Request) {
	wctx := webgo.Context(req)
	wctx.Params // URI paramters
	wctx.Route // is the route configurate itself for which the handler matched
	wctx.AppContext // is the app context configured in `router.AppContext`
}
```

### Helper functions

WebGo has certain helper functions for which the responses are wrapped in a struct according to 
error or successful response. All success responses are wrapped in the struct
```
type dOutput struct {
	Data   interface{} `json:"data"`
	Status int         `json:"status"`
}
```
JSON output would look like
```
{
	"data": <any data>,
	"status: <HTTP response status code, integer>
}
```

All the error responses are wrapped in the struct

```
type errOutput struct {
	Errors interface{} `json:"errors"`
	Status int         `json:"status"`
}
```

JSON output for error response would like

```
{
	"errors": <any data>,
	"status": <HTTP response status code, integer>
}
```

It provides the following helper functions.

1. SendHeader(w http.ResponseWriter, rCode int) - Send only an HTTP response header with the required
response code.
2. Send(w http.ResponseWriter, contentType string, data interface{}, rCode int) - Send any response
3. SendResponse(w http.ResponseWriter, data interface{}, rCode int) - Send response wrapped in
WebGo's default response struct
4. SendError(w http.ResponseWriter, data interface{}, rCode int) - Send an error wrapped in WebGo's
default error response struct
5. Render(w http.ResponseWriter, data interface{}, rCode int, tpl *template.Template) - Render renders
a Go template, with the requierd data & response code.
6. Render404(w http.ResponseWriter, tpl *template.Template) - Renders a static 404 message
7. R200(w http.ResponseWriter, data interface{}) - Responds with the provided data, with HTTP 200
status code
8. R201(w http.ResponseWriter, data interface{}) - Same as R200, but status code 201
9. R204(w http.ResponseWriter) - Sends a reponse header with status code 201 and no response body.
10. R302(w http.ResponseWriter, data interface{}) - Same as R200, but with status code 302
11. R400(w http.ResponseWriter, data interface{}) - Same as R200, but with status code 400 and 
response wrapped in error struct
12. R403(w http.ResponseWriter, data interface{}) - Same as R400, but with status code 403
13. R404(w http.ResponseWriter, data interface{}) - Same as R400, but with status code 404
14. R406(w http.ResponseWriter, data interface{}) - Same as R400, but with status code 406
15. R451(w http.ResponseWriter, data interface{}) - Same as R400, but with status code 451
16. R500(w http.ResponseWriter, data interface{}) - Same as R400, but with status code 500


## Full sample

```
package main

import (
	"net/http"

	"github.com/bnkamalesh/webgo/middlewares"

	"github.com/bnkamalesh/webgo"
)

func helloWorld(w http.ResponseWriter, r *http.Request) {
	wctx := webgo.Context(r)
	webgo.R200(
		w,
		wctx.Params,
	)
}

func getRoutes() []*webgo.Route {
	// var mws webgo.Middlewares

	return []*webgo.Route{
		&webgo.Route{
			Name:     "helloworld",                   // A label for the API/URI, this is not used anywhere.
			Method:   http.MethodGet,                 // request type
			Pattern:  "/",                            // Pattern for the route
			Handlers: []http.HandlerFunc{helloWorld}, // route handler
		},
		&webgo.Route{
			Name:     "helloworld",                                     // A label for the API/URI, this is not used anywhere.
			Method:   http.MethodGet,                                   // request type
			Pattern:  "/api/:param",                                    // Pattern for the route
			Handlers: []http.HandlerFunc{middlewares.Cors, helloWorld}, // route handler
		},
	}
}

func main() {
	cfg := webgo.Config{
		Host:         "",
		Port:         "8080",
		ReadTimeout:  15, // in seconds
		WriteTimeout: 60, // in seconds
	}
	router := webgo.NewRouter(&cfg, getRoutes())
	router.Use(middlewares.AccessLog)
	router.Start()
}

```
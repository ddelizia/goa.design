+++
date="2019-05-09T20:22:44-07:00"
description="github.com/goadesign/goa/middleware/security/basicauth"
+++


# basicauth
`import "github.com/goadesign/goa/middleware/security/basicauth"`

* [Overview](#pkg-overview)
* [Index](#pkg-index)

## <a name="pkg-overview">Overview</a>



## <a name="pkg-index">Index</a>
* [Variables](#pkg-variables)
* [func New(username, password string) goa.Middleware](#New)


#### <a name="pkg-files">Package files</a>
[basicauth.go](/src/github.com/goadesign/goa/middleware/security/basicauth/basicauth.go) 



## <a name="pkg-variables">Variables</a>
``` go
var ErrBasicAuthFailed = goa.NewErrorClass("basic_auth_failed", 401)
```
ErrBasicAuthFailed means it wasn't able to authenticate you with your login/password.



## <a name="New">func</a> [New](/src/target/basicauth.go?s=559:609#L23)
``` go
func New(username, password string) goa.Middleware
```
New creates a static username/password auth middleware.

Example:


	app.UseBasicAuth(basicauth.New("admin", "password"))

It doesn't get simpler than that.

If you want to handle the username and password checks dynamically,
copy the source of `New`, it's 8 lines and you can tweak at will.








- - -
Generated by [godoc2md](http://godoc.org/github.com/davecheney/godoc2md)
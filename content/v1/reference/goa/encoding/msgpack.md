+++
date="2019-05-09T20:22:44-07:00"
description="github.com/goadesign/goa/encoding/msgpack"
+++


# msgpack
`import "github.com/goadesign/goa/encoding/msgpack"`

* [Overview](#pkg-overview)
* [Index](#pkg-index)

## <a name="pkg-overview">Overview</a>



## <a name="pkg-index">Index</a>
* [Variables](#pkg-variables)
* [func NewDecoder(r io.Reader) goa.Decoder](#NewDecoder)
* [func NewEncoder(w io.Writer) goa.Encoder](#NewEncoder)


#### <a name="pkg-files">Package files</a>
[encoding.go](/src/github.com/goadesign/goa/encoding/msgpack/encoding.go) 



## <a name="pkg-variables">Variables</a>
``` go
var (
    Handle codec.MsgpackHandle
)
```
Enforce that codec.Decoder satisfies goa.ResettableDecoder at compile time



## <a name="NewDecoder">func</a> [NewDecoder](/src/target/encoding.go?s=349:389#L19)
``` go
func NewDecoder(r io.Reader) goa.Decoder
```
NewDecoder returns a msgpack decoder.



## <a name="NewEncoder">func</a> [NewEncoder](/src/target/encoding.go?s=473:513#L24)
``` go
func NewEncoder(w io.Writer) goa.Encoder
```
NewEncoder returns a msgpack encoder.








- - -
Generated by [godoc2md](http://godoc.org/github.com/davecheney/godoc2md)
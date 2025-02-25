+++
date = "2020-11-21:01:06-05:00"
title = "Error Handling"
weight = 4

[menu.main]
name = "Error Handling"
parent = "implement"
+++

## Overview

Goa makes it possible to describe precisely the potential errors returned by
service methods. This allows for defining a clear contract between the server
and its clients that gets reflected in the generated documentation and code.

Goa takes a "battery included" approach where errors can be defined with as
little information as just a name. However the DSL also makes it possible to
describe completely new error types when Goa's default is not sufficient.

## Defining Errors

Errors are defined using the
[Error](https://pkg.go.dev/goa.design/goa/v3/dsl#Error) function:

```go
var _ = Service("calc", func() {
    Error("invalid_arguments")
})
```

Errors may also be defined within the scope of a specific method:

```go
var _ = Service("calc", func() {
    Method("divide", func() {
        Payload(func() {
            Field(1, "dividend", Int)
            Field(1, "divisor", Int)
            Required("dividend", "divisor")
        })
        Result(func() {
            Field(1, "quotient", Int)
            Field(2, "reminder", Int)
            Required("quotient", "reminder")
        })
        Error("div_by_zero") // Method specific error
    })
})
```

Both the `invalid_arguments` and `div_by_zero` errors in the examples above
make use of the default error type
[ErrorResult](https://pkg.go.dev/goa.design/goa/v3/expr#ErrorResult).

Custom types can also be used when defining errors as follows:

```go
var DivByZero = Type("DivByZero", func() {
        Description("DivByZero is the error returned when using value 0 as divisor.")
        Field(1, "message", String, "division by zero leads to infinity.")
        Required("message")
})

var _ = Service("calc", func() {
    Method("divide", func() {
        Payload(func() {
            Field(1, "dividend", Int)
            Field(1, "divisor", Int)
            Required("dividend", "divisor")
        })
        Result(func() {
            Field(1, "quotient", Int)
            Field(2, "reminder", Int)
            Required("quotient", "reminder")
        })
        Error("div_by_zero", DivByZero, "Division by zero") // Use DivByZero type to marshal error
    })
})
```

If a type is used to define multiple errors it must define an attribute that
contains the error name so that the generated code can infer the
corresponding design definition. The attribute must be identified via the
`struct:error:name` field
[metadata](https://pkg.go.dev/goa.design/goa/v3/dsl#Meta), for example:

```go
var DivByZero = Type("DivByZero", func() {
    Description("DivByZero is the error returned when using value 0 as divisor.")
    Field(1, "message", String, "division by zero leads to infinity.")
    Field(2, "name", String, "Name of the error", func() {
        // Tell Goa to use the `name` field to match the error definition.
        Meta("struct:error:name")
    })

    Required("message", "name")
})
```

The field must be initialized by the server code that returns the error. The
generated code uses it to match the error definition and compute the proper
status code.

### Temporary Errors, Faults and Timeouts

The `Error` function accepts an optional DSL function as last argument which
makes it possible to specify additional properties on the error. The `Error`
function DSL accepts three child functions:

* `Timeout()` identifies the error as being due to a server timeout.
* `Fault()` identifies the error as a server side fault (e.g. a bug, a panic etc.).
* `Temporary()` identifies the error as being retryable.

The following definition would be appropriate to define a timeout error:

```go
Error("Timeout", ErrorResult, "Request timeout exceeded", func() {
    Timeout()
})
```

The `Timeout`, `Fault` and `Temporary` functions instruct the Goa code
generator to initialize the fields with the same names in the returned
`ErrorResponse` object. They have no effect (other than documentation) when
using a custom error repsonse type.

### Mapping Errors to Transport Status Codes

The [Response](https://pkg.go.dev/goa.design/goa/v3/dsl#Response) function
defines the HTTP or gRPC status code used to write the error:

```go
var _ = Service("calc", func() {
    Method("divide", func() {
        Payload(func() {
            Field(1, "dividend", Int)
            Field(2, "divisor", Int)
            Required("dividend", "divisor")
        })
        Result(func() {
            Field(1, "quotient", Int)
            Field(2, "reminder", Int)
            Required("quotient", "reminder")
        })
        Error("div_by_zero")
        HTTP(func() {
            POST("/")
            Response("div_by_zero", StatusBadRequest, func() { 
                // Use HTTP status code 400 (BadRequest) to write "div_by_zero" errors
                Description("Response used for division by zero errors")
            })
        })
        GRPC(func() {
            Response("div_by_zero", CodeInvalidArgument, func() {
                // Use gRPC status code 3 (InvalidArgument) to write "div_by_zero" errors
                Description("Response used for division by zero errors")
            })
        })
    })
})
```

## Producing Errors

### Using the Default Error Type

With the design defined above Goa generates a helper function `MakeDivByZero`
that the server code may leverage to return errors. The function is generated
in the service specific package (under `gen/calc` in this example). It
accepts a Go error as argument:

```go
// Code generated by goa v....
// ...

package calc

// ...

// MakeDivByZero builds a goa.ServiceError from an error.
func MakeDivByZero(err error) *goa.ServiceError {
	return &goa.ServiceError{
		Name:    "div_by_zero",
		ID:      goa.NewErrorID(),
		Message: err.Error(),
	}
}

// ...
```

This function can be used as follows when implementing the `Divide` function:

```go
func (s *calcsrvc) Divide(ctx context.Context, p *calc.DividePayload) (res *calc.DivideResult, err error) {
    if p.Divisor == 0 {
        return nil, calc.MakeDivByZero(fmt.Errorf("cannot divide by zero"))
    }
    // ...
}
```

The generated `MakeXXX` functions create instances of the
[ServiceError](https://pkg.go.dev/goa.design/goa/v3/pkg#ServiceError) type.

### Using Custom Error Types

When using a user defined type to define errors Goa does not generate helper
functions as it cannot tell how to map Go errors to the user type. Instead
the method implementation should instantiate the error type directly.
Considering the previous example using the `DivByZero` type:

```go
Error("div_by_zero", DivByZero, "Division by zero") // Use DivByZero type to marshal error
```

To return the error the method implementation should return an instance of
the generated `DivByZero` struct also located in the service package (`calc`
in this example):

```go
func (s *calcsrvc) Divide(ctx context.Context, p *calc.DividePayload) (res *calc.DivideResult, err error) {
    if p.Divisor == 0 {
        return nil, &calc.DivByZero{Message: "cannot divide by zero"}
    }
    // ...
}
```

## Consuming Errors

The error values returned to the client are backed by the same structs used by
the server to return errors. 

### Using the Default Error Type

If the error definition uses the default error type then the client side
errors are instances of
[ServiceError](https://pkg.go.dev/goa.design/goa/v3/pkg#ServiceError):

```go
// ... initialize endpoint, ctx, payload
c := calc.NewClient(endpoint)
res, err := c.Divide(ctx, payload)
if res != nil {
    if dbz, ok := err.(*goa.ServiceError); ok {
        // use dbz to handle error
    }
}
// ...
```

### Using Custom Error Types

If the error definition uses a custom type then the client side error is an
instance of the corresponding generated Go struct:

```go
// ... initialize endpoint, ctx, payload
c := calc.NewClient(endpoint)
res, err := c.Divide(ctx, payload)
if res != nil {
    if dbz, ok := err.(*calc.DivByZero); ok {
        // use dbz to handle error
    }
}
// ...
```
 
## Validation Errors

Validation errors are also instances of the
[ServiceError](https://pkg.go.dev/goa.design/goa/v3/pkg#ServiceError) struct.
The `name` field of the struct makes it possible for client code to
differentiate between multiple possible errors.

Here is an example of how to do that which assumes that the design uses the
default error type to define the `div_by_zero` error:

```go
// ... initialize endpoint, ctx, payload
c := calc.NewClient(endpoint)
res, err := c.Divide(ctx, payload)
if res != nil {
    if serr, ok := err.(*goa.ServiceError); ok {
        switch serr.Name {
            case "missing_field":
                // Handle missing operand error
            case "div_by_zero":
                // Handle division by zero error
            default:
                // Handle unknown error
        }
    }
}
// ...
```

The validation error names are all defined in the file 
[error.go](https://github.com/goadesign/goa/blob/v3/pkg/error.go), they are:

* `missing_payload`: error produced when a request is missing a required payload.
* `decode_payload`: error produced when a request body cannot be decoded successfully.
* `invalid_field_type`: error produced when the type of a payload field does not match the type defined in the design.
* `missing_field`: error produced when a payload is missing a required field.
* `invalid_enum_value`: error produced when the value of a payload field does not match one the values defined in the design Enum validation.
* `invalid_format`: error produced the value of a payload field does not match the format validation defined in the design.
* `invalid_pattern`: error produced when the value of a payload field does not match the pattern validation defined in the design.
* `invalid_range`: error produced code when the value of a payload field does not match the range validation defined in the design.
* `invalid_length`: error produced code when the value of a payload field does not match the length validation defined in the design.

## Overridding Error Serialization

Sometimes it may be necessary to override the format used by the generated
code to render validation errors. The generated HTTP handler and server
creation functions make it possible to provide a custom error formatter
function:

```go
// Code generated by goa v...

package server

// ...

// New instantiates HTTP handlers for all the calc service endpoints using the                                                                                                                
// provided encoder and decoder. The handlers are mounted on the given mux                                                                                                                    
// using the HTTP verb and path defined in the design. errhandler is called                                                                                                                   
// whenever a response fails to be encoded. formatter is used to format errors
// returned by the service methods prior to encoding. Both errhandler and                                                                                                                     
// formatter are optional and can be nil.                                                                                                                                                     
func New(                                                                                                                                                                                     
        e *calc.Endpoints,                                                                                                                                                                    
        mux goahttp.Muxer,
        decoder func(*http.Request) goahttp.Decoder,                                                                                                                                          
        encoder func(context.Context, http.ResponseWriter) goahttp.Encoder,                                                                                                                   
        errhandler func(context.Context, http.ResponseWriter, error),                                                                                                                         
        formatter func(err error) goahttp.Statuser,  // Error formatter function
// ...
```

The provided function must accept an instance of an error as argument and
return a struct that implements the
[Statuser](https://pkg.go.dev/goa.design/goa/v3/http#Statuser) interface:

```go
type Statuser interface {
    // StatusCode return the HTTP status code used to encode the response
    // when not defined in the design.
    StatusCode() int
}
```

The generated code calls the struct `StatusCode` method upon writing the HTTP
response and uses the return value to write the HTTP status code. The struct
is then serialized into the response body.

The default implementation used when the value `nil` is given for the
`formatter` argument of the `New` function is
[NewErrorResponse](https://pkg.go.dev/goa.design/goa/v3/http#NewErrorResponse)
which returns an instance of
[ErrorResponse](https://pkg.go.dev/goa.design/goa/v3/pkg#ErrorResponse).

### Overridding Validation Errors Serialization

A custom formatter can inspect the given error value similarly to how client
code does it to format validation errors differently, for example:

```go
// missingFieldError is the type used to serialize missing required field
// errors. It overrides the default provided by Goa.
type missingFieldError string

// StatusCode returns 400 (BadRequest).
func (missingFieldError) StatusCode() int { return http.StatusBadRequest }

// customErrorResponse converts err into a MissingField error if err corresponds
// to a missing required field validation error.
func customErrorResponse(err error) Statuser {
    if serr, ok := err.(*goa.ServiceError); ok {
        switch serr.Name {
            case "missing_field":
                return missingFieldError(serr.Message)
            default:
                // Use Goa default
                return goahttp.NewErrorResponse(err)
        }
    }
    // Use Goa default for all other error types
    return goahttp.NewErrorResponse(err)
}
```

The custom formatter can then be used to instantiate a HTTP server or handler:

```go
var (
    calcServer *calcsvr.Server
)
{
    eh := errorHandler(logger)
    calcServer = calcsvr.New(calcEndpoints, mux, dec, enc, eh, customErrorResponse)
    // ...
```

## Example

The [error](https://github.com/goadesign/examples/tree/master/error) example
illustrates how to use custom error types and how to override the default
error response used for validation errors.


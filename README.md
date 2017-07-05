jwtauth - JWT authentication middleware for Go HTTP services
============================================================

The `jwtauth` http middleware package provides a simple way to verify a JWT token
from a http request and send the result down the request context (`context.Context`).

This package uses the new `context` package in Go 1.7 stdlib and [net/http#Request.Context](https://golang.org/pkg/net/http/#Request.Context) to pass values between handler chains.

In a complete JWT-authentication flow, you'll first capture the token from a http
request, decode it, verify it and then validate that its correctly signed and hasn't
expired - the `jwtauth.Verifier` middleware handler takes care of all of that. The 
`jwtauth.Verifier` will set the context values on keys `jwtauth.TokenCtxKey` and
`jwtauth.ErrorCtxKey`.

Next, it's up to an authentication handler to respond or continue processing after the 
`jwtauth.Verifier`. The `jwtauth.Authenticator` middleware responds with a 401 Unauthorized
plain-text payload for all unverified tokens and passes the good ones through. You can
also copy the Authenticator and customize it to handle invalid tokens to better fit
your flow (ie. with a JSON error response body).

The `Verifier` will search for a JWT token in a http request, in the order:

1. 'jwt' URI query parameter
2. 'Authorization: BEARER T' request header
3. Cookie 'jwt' value
4. (optional), use `jwtauth.Verify("state")` for additional query/cookie parameter aliases

The first JWT string that is found as a query parameter, authorization header
or cookie header is then decoded by the `jwt-go` library and a *jwt.Token
object is set on the request context. In the case of a signature decoding error
the Verifier will also set the error on the request context.

The Verifier always calls the next http handler in sequence, which can either
be the generic `jwtauth.Authenticator` middleware or your own custom handler
which checks the request context jwt token and error to prepare a custom
http response.

# Usage

See the full [example](https://github.com/go-chi/jwtauth/blob/master/_example/main.go).

```go
package main

import (
	"fmt"
	"net/http"

	"github.com/go-chi/chi"
	"github.com/go-chi/jwtauth"
)

var TokenAuth *jwtauth.JwtAuth

func init() {
	TokenAuth = jwtauth.New("HS256", []byte("secret"), nil)

	// For debugging/example purposes, we generate and print
	// a sample jwt token with claims `user_id:123` here:
	_, tokenString, _ := TokenAuth.Encode(jwtauth.Claims{"user_id": 123})
	fmt.Printf("DEBUG: a sample jwt is %s\n\n", tokenString)
}

func main() {
	addr := ":3333"
	fmt.Printf("Starting server on %v\n", addr)
	http.ListenAndServe(addr, router())
}

func router() http.Handler {
	r := chi.NewRouter()

	// Protected routes
	r.Group(func(r chi.Router) {
		// Seek, verify and validate JWT tokens
		r.Use(TokenAuth.Verifier)

		// Handle valid / invalid tokens. In this example, we use
		// the provided authenticator middleware, but you can write your
		// own very easily, look at the Authenticator method in jwtauth.go
		// and tweak it, its not scary.
		r.Use(jwtauth.Authenticator)

		r.Get("/admin", func(w http.ResponseWriter, r *http.Request) {
			_, claims, _ := jwtauth.TokenContext(r.Context())
			w.Write([]byte(fmt.Sprintf("protected area. hi %v", claims["user_id"])))
		})
	})

	// Public routes
	r.Group(func(r chi.Router) {
		r.Get("/", func(w http.ResponseWriter, r *http.Request) {
			w.Write([]byte("welcome anonymous"))
		})
	})

	return r
}
```

# LICENSE

[MIT](/LICENSE)

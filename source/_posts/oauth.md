---
title: GO Oauth2/OIDC客户端/服务器
date: 2018-05-28 09:12:53
tags:
 - golang
categories:
 - 学习笔记
---

1. 服务器参考<https://github.com/RangelReale/osin>
2. 客户端参考<https://github.com/golang/oauth2>
3. OIDC服务器参考<https://github.com/coreos/dex>
4. OIDC客户端参考<https://github.com/coreos/go-oidc>

# OAuth2服务器
``` go
package main

import (
	"fmt"
	"net/http"

	"github.com/RangelReale/osin"
	"github.com/RangelReale/osin/example"
)

func handleLoginPage(ar *osin.AuthorizeRequest, w http.ResponseWriter, r *http.Request) bool {
	r.ParseForm()
	if r.Method == "POST" {
		return true
	}

	html := `<html><body>
LOGIN %s<br/>
<form action="/authorize?%s" method="POST">
<input type="submit"/>
</form>
</body></html>`
	w.Write([]byte(fmt.Sprintf(html, ar.Client.GetId(), r.URL.RawQuery)))
	return false
}

func newTestStorage() *example.TestStorage {
	ts := example.NewTestStorage()
	c, _ := ts.GetClient("1234")
	tc := c.(*osin.DefaultClient)
	tc.RedirectUri = "http://qjw.p.self.kim/callback"
	return ts
}

func main() {
	cfg := osin.NewServerConfig()
	cfg.AllowGetAccessRequest = true
	cfg.AllowClientSecretInParams = true

	server := osin.NewServer(cfg, newTestStorage())

	// Authorization code endpoint
	http.HandleFunc("/authorize", func(w http.ResponseWriter, r *http.Request) {
		resp := server.NewResponse()
		defer resp.Close()

		if ar := server.HandleAuthorizeRequest(resp, r); ar != nil {
			if !handleLoginPage(ar, w, r) {
				return
			}
			ar.Authorized = true
			server.FinishAuthorizeRequest(resp, r, ar)
		}
		if resp.IsError && resp.InternalError != nil {
			fmt.Printf("ERROR: %s\n", resp.InternalError)
		}
		osin.OutputJSON(resp, w, r)
	})

	// Access token endpoint
	http.HandleFunc("/token", func(w http.ResponseWriter, r *http.Request) {
		resp := server.NewResponse()
		defer resp.Close()

		if ar := server.HandleAccessRequest(resp, r); ar != nil {
			ar.Authorized = true
			server.FinishAccessRequest(resp, r, ar)
		}

		if resp.IsError && resp.InternalError != nil {
			fmt.Printf("ERROR: %s\n", resp.InternalError)
		}

		osin.OutputJSON(resp, w, r)
	})

	// Information endpoint
	http.HandleFunc("/info", func(w http.ResponseWriter, r *http.Request) {
		resp := server.NewResponse()
		defer resp.Close()

		if ir := server.HandleInfoRequest(resp, r); ir != nil {
			server.FinishInfoRequest(resp, r, ir)
		}
		osin.OutputJSON(resp, w, r)
	})

	http.ListenAndServe(":14000", nil)
}
```

## OAuth2客户端
``` go
package main

import (
	"fmt"
	"io/ioutil"
	"net/http"

	"golang.org/x/oauth2"
)

const htmlIndex = `<html><body>
<a href="/login">Log in with oauth2</a>
</body></html>
`

var infoUrl = "http://localhost:14000/info"
var endpotin = oauth2.Endpoint{
	AuthURL:  "http://localhost:14000/authorize",
	TokenURL: "http://localhost:14000/token",
}

var oauthConfig = &oauth2.Config{
	ClientID:     "1234",
	ClientSecret: "aabbccdd",
	RedirectURL:  "http://qjw.p.self.kim/callback",
	Scopes:       []string{"api"},
	Endpoint:     endpotin,
}

const oauthStateString = "random"

func main() {
	http.HandleFunc("/", handleMain)
	http.HandleFunc("/login", handleLogin)
	http.HandleFunc("/callback", handleCallback)
	fmt.Println(http.ListenAndServe(":8000", nil))
}

func handleMain(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, htmlIndex)
}

func handleLogin(w http.ResponseWriter, r *http.Request) {
	url := oauthConfig.AuthCodeURL(oauthStateString)
	fmt.Println(url)
	http.Redirect(w, r, url, http.StatusTemporaryRedirect)
}

func handleCallback(w http.ResponseWriter, r *http.Request) {
	state := r.FormValue("state")
	if state != oauthStateString {
		fmt.Printf("invalid oauth state, expected '%s', got '%s'\n", oauthStateString, state)
		http.Redirect(w, r, "/", http.StatusTemporaryRedirect)
		return
	}
	fmt.Println(state)

	code := r.FormValue("code")
	fmt.Println(code)
	token, err := oauthConfig.Exchange(oauth2.NoContext, code)
	fmt.Println(token)
	if err != nil {
		fmt.Println("Code exchange failed with '%s'\n", err)
		http.Redirect(w, r, "/", http.StatusTemporaryRedirect)
		return
	}

	client := &http.Client{}
	req, _ := http.NewRequest("GET", infoUrl, nil)
	req.Header.Set("Authorization", "bearer "+token.AccessToken)
	response, err := client.Do(req)
	if err == nil {
		defer response.Body.Close()
		contents, _ := ioutil.ReadAll(response.Body)
		fmt.Fprintf(w, "Content: %s\n", contents)
	} else {
		fmt.Fprintf(w, "Content: %s\n", err.Error())
	}
}
```

## Gitlab OAuth客户端
``` go
package main

import (
	"fmt"
	"io/ioutil"
	"net/http"

	"golang.org/x/oauth2"
)

const htmlIndex = `<html><body>
<a href="/login">Log in with oauth2</a>
</body></html>
`

//var infoUrl = "http://localhost:14000/info"
//var endpotin = oauth2.Endpoint{
//	AuthURL:  "http://localhost:14000/authorize",
//	TokenURL: "http://localhost:14000/token",
//}
//var oauthConfig = &oauth2.Config{
//	ClientID:     "1234",
//	ClientSecret: "aabbccdd",
//	RedirectURL:  "http://qjw.p.self.kim/callback",
//	Scopes:       []string{"api"},
//	Endpoint:     endpotin,
//}

var domain = "https://gitlab.example.com"
var infoUrl = domain + "/api/v4/users"
var endpotin = oauth2.Endpoint{
	AuthURL:  domain + "/oauth/authorize",
	TokenURL: domain + "/oauth/token",
}
var oauthConfig = &oauth2.Config{
	ClientID:     "4f56sad4f56sa4df65safd",
	ClientSecret: "7f8dasf46sa54f56s4df65",
	RedirectURL:  "http://qjw.p.self.kim/callback",
	Scopes:       []string{"api"},
	Endpoint:     endpotin,
}

const oauthStateString = "random"

func main() {
	http.HandleFunc("/", handleMain)
	http.HandleFunc("/login", handleLogin)
	http.HandleFunc("/callback", handleCallback)
	fmt.Println(http.ListenAndServe(":8000", nil))
}

func handleMain(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, htmlIndex)
}

func handleLogin(w http.ResponseWriter, r *http.Request) {
	url := oauthConfig.AuthCodeURL(oauthStateString)
	fmt.Println(url)
	http.Redirect(w, r, url, http.StatusTemporaryRedirect)
}

func handleCallback(w http.ResponseWriter, r *http.Request) {
	state := r.FormValue("state")
	if state != oauthStateString {
		fmt.Printf("invalid oauth state, expected '%s', got '%s'\n", oauthStateString, state)
		http.Redirect(w, r, "/", http.StatusTemporaryRedirect)
		return
	}
	fmt.Println(state)

	code := r.FormValue("code")
	fmt.Println(code)
	token, err := oauthConfig.Exchange(oauth2.NoContext, code)
	fmt.Println(token)
	if err != nil {
		fmt.Println("Code exchange failed with '%s'\n", err)
		http.Redirect(w, r, "/", http.StatusTemporaryRedirect)
		return
	}

	client := &http.Client{}
	req, _ := http.NewRequest("GET", infoUrl, nil)
	req.Header.Set("Authorization", "bearer "+token.AccessToken)
	response, err := client.Do(req)
	if err == nil {
		defer response.Body.Close()
		contents, _ := ioutil.ReadAll(response.Body)
		fmt.Fprintf(w, "Content: %s\n", contents)
	} else {
		fmt.Fprintf(w, "Content: %s\n", err.Error())
	}
}

```

# OIDC服务器
``` bash
$ go get github.com/coreos/dex
$ cd $GOPATH/src/github.com/coreos/dex
$ make
```

修改examples/config-dev.yml
``` yaml
staticClients:
- id: example-app
  redirectURIs:
  - 'http://qjw.p.self.kim/callback'
  name: 'Example App'
  secret: ZXhhbXBsZS1hcHAtc2VjcmV0
```

``` bash
./bin/dex serve examples/config-dev.yaml
```

## OIDC客户端
``` go
package main

import (
	"encoding/json"
	"log"
	"net/http"

	oidc "github.com/coreos/go-oidc"

	"golang.org/x/net/context"
	"golang.org/x/oauth2"
)

var domain = "http://127.0.0.1:5556/dex"
var (
	clientID     = "example-app"
	clientSecret = "ZXhhbXBsZS1hcHAtc2VjcmV0"
)

func main() {
	ctx := context.Background()

	provider, err := oidc.NewProvider(ctx, domain)
	if err != nil {
		log.Fatal(err)
	}
	oidcConfig := &oidc.Config{
		ClientID: clientID,
	}
	verifier := provider.Verifier(oidcConfig)

	config := oauth2.Config{
		ClientID:     clientID,
		ClientSecret: clientSecret,
		Endpoint:     provider.Endpoint(),
		RedirectURL:  "http://qjw.p.self.kim/callback",
		Scopes:       []string{oidc.ScopeOpenID},
	}

	state := "foobar" // Don't do this in production.

	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		http.Redirect(w, r, config.AuthCodeURL(state), http.StatusFound)
	})

	http.HandleFunc("/callback", func(w http.ResponseWriter, r *http.Request) {
		if r.URL.Query().Get("state") != state {
			http.Error(w, "state did not match", http.StatusBadRequest)
			return
		}

		oauth2Token, err := config.Exchange(ctx, r.URL.Query().Get("code"))
		if err != nil {
			http.Error(w, "Failed to exchange token: "+err.Error(), http.StatusInternalServerError)
			return
		}
		rawIDToken, ok := oauth2Token.Extra("id_token").(string)
		if !ok {
			http.Error(w, "No id_token field in oauth2 token.", http.StatusInternalServerError)
			return
		}
		idToken, err := verifier.Verify(ctx, rawIDToken)
		if err != nil {
			http.Error(w, "Failed to verify ID Token: "+err.Error(), http.StatusInternalServerError)
			return
		}

		oauth2Token.AccessToken = "*REDACTED*"

		resp := struct {
			OAuth2Token   *oauth2.Token
			IDTokenClaims *json.RawMessage // ID Token payload is just JSON.
		}{oauth2Token, new(json.RawMessage)}

		if err := idToken.Claims(&resp.IDTokenClaims); err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		data, err := json.MarshalIndent(resp, "", "    ")
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		w.Write(data)
	})

	log.Fatal(http.ListenAndServe("127.0.0.1:8000", nil))
}
```

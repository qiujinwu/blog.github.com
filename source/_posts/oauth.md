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

# Dex使用Gitlab
Dex默认提供了几种不需要额外配置的认证方式，参见配置 `config-dev.yaml`。代码实现参见`github.com/coreos/dex/storage/static.go`
``` yml
staticClients:
- id: example-app
  redirectURIs:
  - 'http://qjw.p.self.kim/callback'
  name: 'Example App'
  secret: ZXhhbXBsZS1hcHAtc2VjcmV0
  
connectors:
- type: mockCallback
  id: mock
  name: Example
  
# Let dex keep a list of passwords which can be used to login to dex.
enablePasswordDB: true

# A static list of passwords to login the end user. By identifying here, dex
# won't look in its underlying storage for passwords.
#
# If this option isn't chosen users may be added through the gRPC API.
staticPasswords:
- email: "admin@example.com"
  # bcrypt hash of the string "password"
  hash: "$2a$10$2b2cU8CPhOTaGrs1HRQuAueS7JTT5ZHsHSzYiFPm1leZck7Mc8T4W"
  username: "admin"
  userID: "08a8684b-db88-4b73-90a9-3cd1661f5466"
```

为了支持Gitlab，需要
1. 准备gitlab，可以使用gitlab.com，或者自行搭建的私有仓库，用Docker运行Gitlab非常容易，参见<http://blog.self.kim/2017/06/19/gitlab-env/>
2. Gitlab新建`applications`，其中回调<http://server:5556/dex/callback>，*假设dex服务器的地址是http://server:5556,下同*。
3. 修改dex配置,假设自行搭建的gitlab服务器地址<http://server:8080>，注意配置`baseURL`,`redirectURI/clientID/clientSecret`和gitlab上保持一致
``` yml
connectors:
  - type: gitlab
    # Required field for connector id.
    id: gitlab
    # Required field for connector name.
    name: GitLab
    config:
      # optional, default = https://www.gitlab.com 
      baseURL: http://server:8080
      # Credentials can be string literals or pulled from the environment.  
      clientID: 7071eb2135f75bf1d8cabd0c48c98ee38b84e25e628e0f9164cd2e40bd99fcd8
      clientSecret: e84f364add4d221cacf36cfe4136cc1c13872ce1ba17158b5ff972221d9f3e21
      redirectURI: http://server:5556/dex/callback
```

4. 给dex的客户端分配`clientID/clientSecret`，在Gitlab上生成的clientID/clientSecret仅仅供dex自己使用（*站在gitlab角度看，dex是客户端*），分配的方式比较麻烦，dex提供了grpc的接口，参见`github.com/coreos/dex/Documentation/api.md`，编译成功之后有一个`github.com/coreos/dex/bin/grpc-client`。我用了一种比较讨巧的办法先测试，写一个Web API，参见`github.com/coreos/dex/server/server.go`和`github.com/coreos/dex/server/handlers.go`
``` go
handleWithCORS("/token", s.handleToken)
handleWithCORS("/test", s.test)

// 下面的参数`id`和`secret`就是新分配的`clientID/clientSecret`
func (s *Server) test(w http.ResponseWriter, r *http.Request) {
	s.storage.CreateClient(storage.Client{
		ID:           "tt",
		Secret:       "123456",
		RedirectURIs: []string{"http://qjw.p.self.kim/callback"},
		Name:         "hehe",
	})
	w.Write([]byte("Hello, world!"))
}
```

5. 运行客户端
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

var domain = "http://server:5556/dex"
var (
	clientID     = "tt"
	clientSecret = "123456"
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

	log.Fatal(http.ListenAndServe("0.0.0.0:8000", nil))
}
```

## Connector绑定
dex并未就client和特定的connector（比如gitlab）做绑定，若connector只有一个，就直接跳转，否则列出来让用户选

加入dex同时设置了gitlab和github的connector，并且我希望这些clientid只能使用gitlab，另外一些只能使用github做授权，就无法实现。
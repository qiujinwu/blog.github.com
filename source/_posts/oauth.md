---
title: oauth2客户端和服务器sample
date: 2018-05-28 09:12:53
tags:
 - golang
categories:
 - 学习笔记
---

服务器参考<https://github.com/RangelReale/osin>

# 服务器
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

## 客户端
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

## Gitlab
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


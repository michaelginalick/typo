---
title: "Go builder pattern"
date: "2024-08-08"
summary: "A simple use case for functional options"
description: "A builder pattern hidden behind functional options"
toc: false
readTime: true
autonumber: true
math: false
tags: ["builder pattern", "Go", "functional options"]
showTags: true
hideBackToTop: false
---

After spending the majority of my developer career using Ruby and Ruby on Rails, I was looking for a chance to expand my technical apptitude. Golang had been of interest to me for a few years. The majority of my side projects are written in Go, but I did not have any professional experience. Fortunately for me, an opportunity presented itself working primarily with Go. It was a welcome change, but did not come without it's struggles. Maybe it's just me, but after spending so much time with one language, I started to only think about problems in the scope of that particular language. Learning is often difficult and shifting to a new programming language is no exception. However, I digress.

One of the first features I was tasked to implement in Go was a simple API client. A way to consume a relatively simple REST server was needed by various applications. Albeit simple enough, I now know the exact wrong thing to do. The first iteration was to have the consumer of this package know as little as possible about the implementation. At the time, there was a single API key to set as a header, an auth-token header, a base url, and one or two other simple things. Great! I thought to myself. I can make a single API method that gives back an instance of a request with all the necessary auth and necessary parameters set. The implementation looked very similar to this:

```go
package main

import (
  apiClient "www.github.com/me/example/apiclient"
)
func main() {
  // stuff before
  csRequest := apiClient.DefaultDeliveryClient()
  // stuff after
}

```
I thought to myself, this works well. Now the consumer just needs to know about one method and the package will handle everything. All the complications in setting up the request are isolated inside of the package, and none of the details are exposed. On the other side however, is where things started to go wrong. Dispite having having a relatively easy to use API, the way it was set up was largely hardcoded. The implementation looked something like this:

```golang

func NewEntryClientWithDefaults() (HTTPClient, error) {
	config, err := BuildWithEntryConfig()
	if err != nil {
		return nil, err
	}

	if config.ApiKey == "" || config.Token == "" {
		return nil, ErrMissingCredentials
	}

	return &ContentstackClient{
		requestBuilder: &reqBuilder.RequestBuilder{
			Timeout: DefaultTimeout,
			Config:  &config,
		},
	}, nil

}

func BuildWithEntryConfig() (reqBuilder.Config, error) {
	v := viper.New()
	v.SetEnvPrefix("contentstack")
	v.AutomaticEnv()

	err := v.ReadConfig(bytes.NewBuffer([]byte("")))
	if err != nil {
		return reqBuilder.Config{}, err
	}
	v.SetDefault("default_base_url", DefaultBaseURL)
	v.SetDefault("default_resource", DefaultResource)
	v.SetDefault("default_api_version", DefaultApiVersion)

	return reqBuilder.Config{
		Token:       v.GetString("access_token"),
		ApiKey:      v.GetString("api_key"),
		Environment: v.GetString("environment"),
		Resource:    v.GetString("default_resource"),
		BaseURL:     v.GetString("default_base_url"),
		ApiVersion:  v.GetString("default_api_version"),
	}, nil
}
```

The specifics are largely unimportant save for the BuildWithEntryConfig function. Notice how a Config struct is being used. This means, that the implementation is very restrictive. Now, if the business never changed this would have been fine. This worked for for about a year, until of course further integrations needed to be done. Now I had a problem, I could replicate the same Config pattern with another function that is identical except for the config values being different. This option was ripe with code smell. All the time I was thinking to myself, if this was Ruby, I could utilize the builder pattern so the client could specify the properties thier instance needs. I was invisioning the implementation would look something like this:

```ruby
	def with_access_token(access_token)
		self.access_token = access_token
		self
	end

	def with_api_version(api_version)
		self.api_version(api_version)
		self
	end

```

After some poking around on the internet I found what I was looking for. In a [post](https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis) that is over 10 years old, I found myself reading my exact experience. While humbling, it was perfect. To me, it looks like the builder pattern with a slighly different name and syntax, but the principles are the same. Now, I can rewrite my API to allow the clients to set what they want. The implementation on the consumer side now looks like:

```go

import (
	csClient "github.com/me/example"
)

func DefaultContentDeliveryClient() Client {
	return NewBuilderClient(
		csClient.WithAccessToken(os.Getenv("ACCESS_TOKEN")),
		csClient.WithAccessTokenHeaderKey(os.Getenv("API_KEY")),
		csClient.WithBaseUrl("https://example.io"),
	)
}
```

On the API side, the implementation looks something like this:

```go
type ContentstackRequest struct {
	Request *http.Request
}

type Option func(*ContentstackRequest) error

func New(opts ...Option) (*ContentstackRequest, error) {
	client := &ContentstackRequest{
		Request: &http.Request{
			Header: make(http.Header),
			URL:    &url.URL{},
		},
	}
	for _, opt := range opts {
		if err := opt(client); err != nil {
			return &ContentstackRequest{}, fmt.Errorf("option failed %w", err)
		}
	}

	return client, nil
}

func WithApiKeyHeaderKey(apiKeyHeader string) func(*ContentstackRequest) error {
	return func(client *ContentstackRequest) error {
		client.Request.Header.Add(ApiKeyHeader, apiKeyHeader)
		return nil
	}
}

func WithAccessTokenHeaderKey(accessToken string) func(*ContentstackRequest) error {
	return func(client *ContentstackRequest) error {
		client.Request.Header.Add(AuthTokenHeader, accessToken)
		return nil
	}
}

func WithBaseUrl(baseUrl string) func(*ContentstackRequest) error {
	return func(client *ContentstackRequest) error {
		parsedUrl, err := url.ParseRequestURI(baseUrl)
		if err != nil {
			return err
		}
		client.Request.URL = parsedUrl
		return nil
	}
}

```
In my opinion, this is a much better implementation. The API is much more flexible without exposing any of the implementation details. While it may not be a one to one match, I interpret this as a builder pattern.

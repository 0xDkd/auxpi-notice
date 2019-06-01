+++
title = "关于 auxpi 对接 flickr 图床的一些坑"
author = "aimer"
github_url = ""
head_img = "https://test.demo-1s.com/dispatch/3eef09272b0d6d66d20878283d676a90"
created_at = 2019-06-01T18:05:25
updated_at = 2019-06-01T18:05:25
description = "flickr 的认证真的是坑坑坑啊"
tags = ["remote", "go", "auxpi"]
+++

![flickr](https://test.demo-1s.com/dispatch/3eef09272b0d6d66d20878283d676a90)

> 在开发 auxpi 图床的时候踩了不少坑，特别是用 flickr 的时候，简直调试到内心崩溃，我再 github 上找了一个 flickr 的第三方SDK，Go 的 SDK 并不多，而且说明很少，只能去看源码。那种简直看到心态爆炸 QAQ


## multipart.File to os.File 
使用 beego上传文件的时候，默认接受到的文件格式是 multipart.File ，但是我发现第三方 Flickr 的 SDK所提供的文件上传类型其实是 os.File 的类型。他是用一个 `io.Reader` 做为参数传入进去的。通过阅读源代码


```go
func UploadFile(client *FlickrClient, path string, optionalParams *UploadParams) (*UploadResponse, error) {
	file, err := os.Open(path)
	if err != nil {
		return nil, err
	}
	defer file.Close()
	return UploadReader(client, file, file.Name(), optionalParams)
}
```

我们可以发现，这个 `io.Reader ` 其实是一个 `os.File` 的指针类型，即为 `*os.File` 我们可以直接调用 `UploadReader` 这个函数，直接将已经构造好的 `client`传入进去就可以或获得结果。

那么问题来了，如何将 `multipart.File` 类型转换为 `os.File` 类型,其实相当简单


```go
multipartFile := *multipart.FileHeader
osFile :=multipartFile.Open()
```
`osFile` 就是我们需要的 `os.File` 类型

## Oauth Flickr

其实这个玩意是最值得吐槽的，Flickr 的授权简直了，太恶心人了，真的就像是一坨💩，这是我用过最恶心人的授权。

弄明白它的授权需要了解下面几个比较重要的东西

* URL
* Method
* Api_key
* Api_secret
* Oauth_token
* Oauth_token_secret
* ID

在构造 `Client` 的时候，`Client` 应该具有上面的字段(不包括 url)，否则就会出现各种恶心的报错。

其中  URL 需要签名，这个就需要参照 `Oauth1.0` 的要求去看啦，不过这玩意用起来也太恶心了……

第三方 SDK 在这一点做的挺不错的，具体如何实现的可以去阅读以下源代码

## Flickr SDK Update

这里需要给这个第三方的 SDK 更新一点东西，我发现他的生成出来的 `Client` 模式的权限是 `delete` 而我们所需要的权限是 `write` 或者其他的权限，所以说权限最好可以自己设置，所以修改了一下他的代码



最后写了一个 Flickr 授权的一个命令行工具，有兴趣的可以看一下，就几十行代码，这个工具可以对接 `AUXPI` 图床进行使用


```go
package main

import (
	"fmt"

	flickr "gopkg.in/masci/flickr.v2"
)

var (
	api_key    string
	api_secret string
	method     string
	frob       string
)

func main() {
	oauth()
}

func oauth() {
	fmt.Println("Please input your API key ")
	fmt.Scanln(&api_key)
	fmt.Println("please input your API Secret ")
	fmt.Scanln(&api_secret)
	client := flickr.NewFlickrClient(api_key, api_secret)
	requestok, _ := flickr.GetRequestToken(client)
	url, err := flickr.GetAuthorizeUrl(client, requestok, "write")
	if err != nil {
		fmt.Println("Please Check Your Net Work,I can not connect to flickr")
		return
	}
	fmt.Println("")
	fmt.Println("Please Open the follow url to get your code!")
	fmt.Println("")
	fmt.Println(url)
	var code string
	fmt.Println("Please input your Code:")
	fmt.Scan(&code)
	fmt.Println("<===Done===>")
	_, errs := flickr.GetAccessToken(client, requestok, code)
	if errs != nil {
		fmt.Println("Your API_KEY or API_SECRET or Your Code is error")
		return
	}
	fmt.Println("<==Please Input these value to your SiteConfig.json==>")
	fmt.Println("oauth_token:", client.OAuthToken)
	fmt.Println("oauth_token_secret:", client.OAuthTokenSecret)
	fmt.Println("oauth_id:", client.Id)
}
```

获取到的 `oauth_token`,`oauth_token_secret`,`oauth_id` 分别填入到 `SiteConfig.json` 里面，AUXPI 图床就可以快乐的使用 flickr 啦。





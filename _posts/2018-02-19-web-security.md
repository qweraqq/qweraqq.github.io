---
layout: post
title: "Web安全蓝图"
date: 2018-02-19 00:00:00 +0800
author: xiangxiang
categories: web security
tags: [web security]
---
从Web发展的历史看安全

## Part 1: HTTP basics
### 1. HTTP request header format
{% highlight text %}
VERB /resource/locator HTTP/1.1
Header1:Value1
Header2:Value2
...

<Body of request>
{% endhighlight %}

### 2. HTTP request headers
+ Host
+ Accept
    + MIME type(s) are accepted by the client
    + often used to specify JSON or XML output for web-services
+ Cookie [RFC6265](https://tools.ietf.org/html/rfc6265)
+ Referer: Page leading to this request
    + not passed to other servers when using HTTPS on the origin
+ Authorization: 'basic auth' pages(mainly)
    + takes the form **"Basic <base64'd username:password>"**

### 3. cookie: key-value pairs of data with scope(**cookie domain**)
+ [cookie RFC6265](https://tools.ietf.org/html/rfc6265)
+ two important cookie security flag:
    - **Secure**: the cookie will only be accessible to HTTPS pages
    - **HTTPOnly**: the cookie cannot be read by JS
    - the server indicates these flags in the Set-Cookie header that passes them **in the first place**

### 4. HTML:  **parsed** according to the relevant spec(generally HTML5 now);
+ legacy parsing (bad HTML,missing tag)
+ MIME sniffing (aka [content sniffing]((https://en.wikipedia.org/wiki/Content_sniffing))) 
+ encoding sniffing
    + Notes: often not just parsed by browser
    + but also Web-Application Firewalls and other filters

### 5. Same-Origin Policy: protocol+host+port
+ protocol: http/https
+ host
    + http:// *en.example.com* /XX 
    + http:// *www.example.com* /XX 
    + -> different hosts
+ port
+ SOP relaxing:
    + cross-origin resource sharing(CORS)
    + document.domain property
    + cross-document messaging
    + JSONP
    + WebSockets

## Part 2: Web security outline
### 1. Web 1.0
{% highlight text %}
            HTTP 
client  <--------->  server
    |                   |
[browser]          [Web server]
    |                   | 
    |                   |
(private)            database 
   data
{% endhighlight %}

#### Attacks
+ **SQL injection** (combine code&data, similar to buffer overflow)

+ The web with state(HTTP is stateless) -> hidden fields/cookies
    + **session hijacking**
    + **cross-site scripting forgery** (CSRF)

### 2. Web 2.0
JavaScript cant read/write cookies & alter DOM -> Same-Origin Policy(SOP)

#### Attacks
+ **Cross-site scripting** (XSS) -> subverting SOP

## Part 3: An Overview of HTTP Security Headers
### 1. [Content-Security-Policy](https://developer.mozilla.org/en-US/docs/Web/Security/CSP)
+ allow the owners of a web application to inform the client browser about expected behaviour of the application
+ including content sources, script sources, plugin types, and other remote resources
+ allows the browser to more intelligently enforce security constraints
+ example
```
Content-Security-Policy: default-src ‘self’; img-src *; object-src media1.example.com media2.example.com *.cdn.example.com; script-src trustedscripts.example.com
```
+ A list of these directives can be found below, note these are not all of them but the most popular ones

|  directives    | explanation                                           |
|  ------------  | ----------------------------------------------------  |
| default-src    | This acts as a catchall for everything else.          |
| script-src     | Describes where we can load javascript files from     |
| style-src      | Describes where we can load stylesheets from          |
| img-src        | Describes where we can load images from               |
| connect-src    | Applies to AJAX and Websockets                        |
| font-src       | Describes where we can load fonts from                |
| script-src     | Describes where we can load javascript files from     |
| object-src     | Describes where we can load objects from (<embed>)    |
| media-src      | Describes where we can load audio and video files from|
| frame-ancestors| Describes which sites can load this site in an iframe |

+ This source list can be found below:

|  source               | explanation                                           |
|  ------------         | ----------------------------------------------------  |
| *                     | Load resources from anywhere                          |
| 'none'                | Block everything                                      |
| 'self'                | Can only load resources from same origin              |
| data:                 | Can only load resources from data schema (Base64)     |
| something.example.com | Can only load resources from specified domain         |
| https:                | Can only load resources over HTTPS                    |
| 'unsafe-inline'       | Allows inline elements (onclick,<script></script> tags, javascript:,)   |
| 'unsafe-eval'         | Allows dynamic code evaluation (eval() function)       |
| 'sha256-'             | Can only load resources if it matches the hash         |
| 'nonce-'              | Allows an inline script or CSS to execute if the script tag contains a nonce attribute matching the nonce specifed in the CSP header. |

### 2. [Strict-Transport-Security](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Strict-Transport-Security)
+ force browsers to only communicate with the server over a secure connection
+ it is commonly set to a “max-age” value high enough to ensure the website is cached in the HSTS list for the defined period
```
Strict-Transport-Security: max-age=31536000; 
```

### 3. [X-Frame-Options](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options)
+ iframe and [clickjacking](https://en.wikipedia.org/wiki/Clickjacking)
+ In order to avoid this, the “X-Frame-Options” header was created
+ lets the owner of the website decide which sites are allowed to frame their site
+ "X-Frame-Options" header has been deprecated and will be replaced by the Frame-Options directive in the Content Security Policy
```
X-Frame-Options: SAMEORIGIN 
```

### 4. [X-XSS-Protection](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-XSS-Protection)
+ Modern browsers include a feature to help prevent against reflected cross-site scripting attacks, known as the XSS Filter
+ The "X-XSS-Protection" header can be used to enable or disable this built-in feature
```
X-XSS-Protection: 1; mode=block. 
```

### 5. [X-Content-Type-Options](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Content-Type-Options)
+ A nifty attack known as MIME type confusion was the reason this header was created
+ [MIME sniffing](https://en.wikipedia.org/wiki/Content_sniffing)
+ "X-Content-Type-Options" can be used to prevent this “educated” guess from happening by setting the value of this header to "nosniff"
```
X-Content-Type-Options: nosniff 
```

### 6. [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS )
+ It is common to find websites that interact with other websites, for example requesting resources such as JavaScript scripts and fonts
+ cross-origin resource sharing (CORS) that makes this possible in a secure manner
+ This header is used to determine which websites are allowed to access certain resources.
+ The use of a wildcard (*) is allowed, but not generally recommended.
+ As an example of this header, to permit https://www.example.com to request and integrate your site’s data and content would be the following
```
Access-Control-Allow-Origin: https://www.example.com
```


实际中选择一个好的框架(secure by default)，按照文档好好学习。

#### 参考链接
1. [An Overview of HTTP Security Headers](https://www.dionach.com/en-us/blog/an-overview-of-http-security-headers/)
2. [What is the difference between CORS and CSPs](https://stackoverflow.com/questions/39488241/what-is-the-difference-between-cors-and-csps)
3. [Content Security Policy (CSP) Bypasses](http://ghostlulz.com/content-security-policy-csp-bypasses/)
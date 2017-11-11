---
layout: post
title: "Hypertext Transfer Protocol -- HTTP/1.1"
date: 2017-11-06 22:45:00 +0800
categories: rfc
tags:
- rfc
- http
---

超文本传输协议是互联网上应用最为广泛的一种网络协议，它演化了很多版本，第一个有记载的版本[HTTP/0.9](https://www.w3.org/Protocols/HTTP/AsImplemented.html)被设计用于从服务器获取HTML文档。1996年，[HTTP/1.0](https://tools.ietf.org/html/rfc1945)由 [Tim Berners-Lee](https://en.wikipedia.org/wiki/Tim_Berners-Lee) 等发布为[RFC1945](https://tools.ietf.org/html/rfc1945)，这个版本具有划时代的意义。目前最常见的HTTP协议版本为[HTTP/1.1](https://tools.ietf.org/html/rfc2616)，于1999年发布，之后有一系列RFC对它进行更新和细化。本文记录笔者阅读[RFC2616](https://tools.ietf.org/html/rfc2616)过程中的要点。

超文本传输​​协议（HTTP）是分布式协作超媒体信息系统的应用级协议。它是一个通用的、无状态的协议，通过对请求方法、错误代码和头信息的扩展，可将它用于超文本之外的许多任务，例如 Name Server 和分布式对象管理系统。自1990年起，HTTP一直被环球信息网(WWW)使用。

**术语 Terminology**
```
connection(连接):
  为了通信而在两个程序之间建立的传输层虚拟电路

message(消息):
  HTTP通信的基本单元，由与 HTTP Message 一节中定义的语法相匹配的结构化字节序列组成，并通过连接进行传输

request(HTTP请求消息)

response(HTTP响应消息)

resource(资源):
  可通过URI进行标识的网络数据对象或服务。资源可由多种表示形式（如多种语言、数据格式、大小和分辨率）提供

entity(实体):
  作为请求或响应的有效载荷(payload)传输的信息。实体由entity-header形式的元信息和entity-body形式的内
  容组成

representation(表示):
  包含在内容协商响应中的实体，可能存在与特定响应状态相关联的多个表示

content negotiation(内容协商):
  服务请求时选择适当表示的机制。任何响应中的实体的表示都可以协商(包括错误响应)

variant(变体):
  资源可以在任何给定的时刻具有与其关联的一个或多个表示。这些表示中的每一个都被称为“varriant”。使用术语
  “变体”并不一定意味着资源需要进行内容协商

client(客户端):
  为发送请求而建立连接的程序

user agent(用户代理):
  发起请求的客户端。通常是浏览器、编辑器、网络爬虫或其它用户工具

server(服务器):
  通过发送响应来接受连接以服务请求的应用程序。任何服务器都可以充当源服务器、代理、网关或隧道，根据每个
  请求的性质来切换行为

origin server(源服务器):
  资源驻留或将要在其之上创建的服务器

proxy(代理):
  服务器和客户端的中间程序，对客户端来说，它是服务器，同事代表客户端发送请求，作为远端服务器的客户端。代
  理分为透明代理和非透明代理

gateway(网关):
  与代理不同，网关接收请求，就好像它是请求资源的源服务器一样。请求客户端可能不知道它正在与网关进行通信

tunnel(隧道):
  作为两个连接之间的盲中继。一旦激活，隧道不被认为是HTTP通信的一方，尽管隧道可能已经由HTTP请求发起。当
  中继连接的两端关闭时，隧道不再存在

cache(缓存):
  程序的响应消息的本地存储区和控制其消息存储，检索和删除的子系统

cacheable:
  如果允许缓存存储响应消息的副本以用于应答后续请求，则响应是可缓存的。隧道消息不可缓存。

first-hand:
  如果响应直接来自源服务器，并且没有代理服务器进行不必要的延迟，那么响应是第一手的；如果直接与源服务器检
  查其有效性，则回应也是第一手的

explicit expiration time(明确的到期时间):
  源服务器希望实体不应再由缓存返回的时间，无需进一步验证

heuristic expiration time(启发式到期时间):
  当没有明确的到期时间可用时，由缓存分配的到期时间

age:
  响应的年龄是它由源服务器发送或成功验证之后经历的时间

freshness lifetime:
  响应的生成和到期之间的时间长度

fresh:
  一个fresh的响应满足: age <= freshness lifetime

stale:
  一个stale的响应满足: age > freshness lifetime

semantically transparent(语义透明):
  当缓存的使用既不影响请求客户端也不影响源服务器时(除了提高性能)，缓存的行为就是一种“语义透明”的方式

validator:
  用于找出缓存条目是否是实体的等效副本的协议元素（例如，实体标签或上次修改时间）

upstream/downstream:
  上游和下游描述消息流: 所有消息从上游流向下游

inbound/outbound:
  入站和出站是指消息的请求和响应路径: “入站”是指“前往源服务器”，“出站”是指“前往用户代理”
```

HTTP协议是一个请求/响应协议。简单的HTTP连接:

```
       request chain ------------------------>
    UA -------------------v------------------- O
       <----------------------- response chain

UA -> User Agent; O -> Origin Server; v -> connection
```

复杂的HTTP连接可能有多个中间程序（如Proxy、gateway、tunnel）:

```
       request chain -------------------------------------->
    UA -----v----- A -----v----- B -----v----- C -----v----- O
       <------------------------------------- response chain
```

任何非隧道通信方都可以使用内部缓存来处理请求，缓存的作用是，如果链中的一个参与者具有适用于该请求的缓存响应，则请求/响应链将被缩短:

```
B 缓存了来自 C 的响应

       request chain ---------->
    UA -----v----- A -----v----- B - - - - - - C - - - - - - O
       <--------- response chain
```

HTTP通信通常通过TCP/IP连接进行。默认端口是TCP 80，但也可以使用其它端口。这并不排除HTTP在互联网或其它网络上的其它协议的基础上实施。HTTP只是假设一个可靠的传输，任何提供这种保证的协议都可以使用。

HTTP/1.0的大多数实现为每个请求/响应交换使用一个新的连接。在HTTP/1.1中，一个连接可以用于一个或多个请求/响应交换，尽管由于各种原因连接可能会关闭。

<h4>协议参数</h4>

**HTTP Version**

HTTP使用“<major>.<minor>”编号方案来指示协议的版本。协议版本策略旨在允许发送者指示消息的格式及其用于理解进一步HTTP通信的能力。应用程序的HTTP版本是应用程序至少符合条件的最高HTTP版本。

代理和网关应用程序在使用不同于应用程序的协议版本转发消息时需要小心。由于协议版本指示了发送者的协议能力，所以代理/网关绝不能发送大于其实际版本的消息。如果接收到更高版本的请求，则代理/网关必须降级请求版本，或响应错误，或切换到隧道行为。

**Uniform Resource Identifiers(统一资源标识符)**

URI具有很多名称: WWW地址、通用文档标识符、通用资源标识符(Universal Resource Identifiers)，最后是统一资源定位符(URL)和名称(URN)的组合。就HTTP而言，统一资源标识符是简单的格式化字符串，通过名称，位置或任何其它特性来标识资源。URI细节可以查看 [RFC 2396](https://tools.ietf.org/html/rfc2396) 。

```
  http_URL = "http:" "//" host [ ":" port ] [ abs_path [ "?" query ]]
```

如果端口是空的或者没有给出，则假定端口80。如果在路径中不存在abs_path，那么在用作资源的Request-URI时，它必须以“/”的形式给出。

**Date/Time Formats**

```
       HTTP-date    = rfc1123-date | rfc850-date | asctime-date
       rfc1123-date = wkday "," SP date1 SP time SP "GMT"
       rfc850-date  = weekday "," SP date2 SP time SP "GMT"
       asctime-date = wkday SP date3 SP time SP 4DIGIT
       date1        = 2DIGIT SP month SP 4DIGIT
                      ; day month year (e.g., 02 Jun 1982)
       date2        = 2DIGIT "-" month "-" 2DIGIT
                      ; day-month-year (e.g., 02-Jun-82)
       date3        = month SP ( 2DIGIT | ( SP 1DIGIT ))
                      ; month day (e.g., Jun  2)
       time         = 2DIGIT ":" 2DIGIT ":" 2DIGIT
                      ; 00:00:00 - 23:59:59
       wkday        = "Mon" | "Tue" | "Wed"
                    | "Thu" | "Fri" | "Sat" | "Sun"
       weekday      = "Monday" | "Tuesday" | "Wednesday"
                    | "Thursday" | "Friday" | "Saturday" | "Sunday"
       month        = "Jan" | "Feb" | "Mar" | "Apr"
                    | "May" | "Jun" | "Jul" | "Aug"
                    | "Sep" | "Oct" | "Nov" | "Dec"
```

**Character Sets(字符集)**

本文档中使用的术语“字符集”是指与一个或多个表一起使用以将八位字节序列转换为字符序列的方法。

**Content Codings(内容编码)**

内容编码的值用于指示已经或可以应用于实体的编码转换。内容编码主要用于允许压缩文档或进行其它有用的转换。HTTP/1.1在`Accept-Encoding`和`Content-Encoding`头域中使用内容编码值。互联网号码分配机构（IANA）作为内容编码值的注册机构。最初，注册表包含以下标记:

```
    gzip        An encoding format produced by the file compression program "gzip" as
                described in RFC 1952
    compress    The encoding format produced by the common UNIX file compression
                program "compress"
    deflate     The "zlib" format defined in RFC 1950
    identity    The default (identity) encoding; the use of no transformation whatsoever.
                This content-coding is used only in the Accept-Encoding header
```

**Transfer Codings(传输编码)**

传输编码值用于指示已经，可能或可能需要应用于实体主体的编码转换，以确保通过网络的“安全传输”。这与内容编码的不同之处在于，传输编码是消息的属性，而不是原始实体的属性。

传输编码类似于MIME的Content-Transfer-Encoding值，它被设计用于在7位传输服务上安全地传输二进制数据。

**Media Types(媒体类型)**

HTTP使用`Content-Type`和`Accept`头字段中的Internet媒体类型来提供开放和可扩展的数据类型和类型协商。

```
    media-type     = type "/" subtype *( ";" parameter )
    type           = token
    subtype        = token
```

MIME提供了许多`multipart`类型 —— 在单个消息体中封装一个或多个实体。所有`multipart`类型共享一个通用语法(见 [RFC 2046](https://tools.ietf.org/html/rfc2046#section-5.1.1))，并且必须包含边界参数作为媒体类型值的一部分。

**Product Tokens**

用于允许通信应用程序通过软件名称和版本来标识自己，如:

```
    User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_3) AppleWebKit/537.36
    (KHTML, like Gecko) Chrome/62.0.3202.75 Safari/537.36
```

**Quality Values**

HTTP内容协商使用短“浮点”数字来表示各种可协商参数的相对重要性（“权重”）。权重为0到1范围内的实数，其中0是最小值，1是最大值。如果参数的质量值为0，则该参数的内容对于客户端是“不可接受的”。 HTTP/1.1应用程序不能在小数点后生成三位以上的数字。这些值的用户配置也应该以这种方式进行限制。

```
    qvalue         = ( "0" [ "." 0*3DIGIT ] )
                   | ( "1" [ "." 0*3("0") ] )
```

**Language Tags**

语言标签标识人类口头的、书面的或以其它方式传达的用于将信息传递给他人的自然语言。HTTP使用`Accept-Language`和`Content-Language`字段中的语言标签。HTTP语言标签的语法和注册与 [RFC 1766](https://tools.ietf.org/html/rfc1766) 中的定义相同。

```
    language-tag  = primary-tag *( "-" subtag )
    primary-tag   = 1*8ALPHA
    subtag        = 1*8ALPHA

eg:
    en, en-US, en-cockney, i-cherokee, x-pig-latin
```

**Entity Tags**

实体标签用于比较来自相同请求资源的两个或多个实体。HTTP/1.1在`ETag`，`If-Match`，`If-None-Match`和`If-Range`头字段中使用实体标签。

```
    entity-tag = [ weak ] opaque-tag
    weak       = "W/"
    opaque-tag = quoted-string
```

**Range Units**

HTTP/1.1允许客户端请求只包含响应实体的一部分（一定范围)的响应。HTTP/1.1使用`Range`和`Content-Range`头字段。

```
    range-unit       = bytes-unit | other-range-unit
    bytes-unit       = "bytes"
    other-range-unit = token
```

<h4>HTTP Message</h4>

不得不提一句，扩展的巴科斯范式([RFC2234](https://tools.ietf.org/html/rfc2234))真是太清晰了。刚看的时候可能不习惯，时间长了真的非常容易理解。


消息类型:
```
    HTTP-message   = Request | Response     ; HTTP/1.1 messages
```

消息格式(含Request和Response):
```
    generic-message = start-line
                      *(message-header CRLF)
                      CRLF
                      [ message-body ]
    start-line      = Request-Line | Status-Line
```

消息头(general-header&request-header|response-header):
```
    message-header  = field-name ":" [ field-value ]
    field-name      = token
    field-value     = *( field-content | LWS )
    field-content   = <the OCTETs making up the field-value
                      and consisting of either *TEXT or combinations
                      of token, separators, and quoted-string>
```

消息正文:
```
    message-body = entity-body
                 | <entity-body encoded as per Transfer-Encoding>
```

**General Header Fields**

对请求和响应消息具有普遍适用性的头字段:

```
    general-header  = Cache-Control
                    | Connection
                    | Date
                    | Pragma
                    | Trailer
                    | Transfer-Encoding
                    | Upgrade
                    | Via
                    | Warning
```

**请求消息**

```
    Request       = Request-Line
                    *(( general-header
                     | request-header
                     | entity-header ) CRLF)
                    CRLF
                    [ message-body ]

    Request-Line   = Method SP Request-URI SP HTTP-Version CRLF

    Method         = "OPTIONS"
                      | "GET"
                      | "HEAD"
                      | "POST"
                      | "PUT"
                      | "DELETE"
                      | "TRACE"
                      | "CONNECT"
                      | extension-method
    extension-method = token

    Request-URI    = "*" | absoluteURI | abs_path | authority

    注: 由Internet请求标识的确切资源是通过检查Request-URI和Host头字段来确定的。向代理发出请求时，
        必须填写absoluteURI，此时Host头字段被忽略。

    request-header    = Accept
                      | Accept-Charset
                      | Accept-Encoding
                      | Accept-Language
                      | Authorization
                      | Expect
                      | From
                      | Host
                      | If-Match
                      | If-Modified-Since
                      | If-None-Match
                      | If-Range
                      | If-Unmodified-Since
                      | Max-Forwards
                      | Proxy-Authorization
                      | Range
                      | Referer
                      | TE
                      | User-Agent

    request-header允许客户端将关于请求的附加信息以及客户端本身的附加信息传递给服务器。

    general-header 和 request-header 之外的头字段为 entity-header。
```

**响应消息**

```
    Response      = Status-Line
                    *(( general-header
                     | response-header
                     | entity-header ) CRLF)
                    CRLF
                    [ message-body ]

    Status-Line = HTTP-Version SP Status-Code SP Reason-Phrase CRLF

    Status-Code     =
                      "100"  ; Continue
                    | "101"  ; Switching Protocols
                    | "200"  ; OK
                    | "201"  ; Created
                    | "202"  ; Accepted
                    | "203"  ; Non-Authoritative Information
                    | "204"  ; No Content
                    | "205"  ; Reset Content
                    | "206"  ; Partial Content
                    | "300"  ; Multiple Choices
                    | "301"  ; Moved Permanently
                    | "302"  ; Found
                    | "303"  ; See Other
                    | "304"  ; Not Modified
                    | "305"  ; Use Proxy
                    | "307"  ; Temporary Redirect
                    | "400"  ; Bad Request
                    | "401"  ; Unauthorized
                    | "402"  ; Payment Required
                    | "403"  ; Forbidden
                    | "404"  ; Not Found
                    | "405"  ; Method Not Allowed
                    | "406"  ; Not Acceptable
                    | "407"  ; Proxy Authentication Required
                    | "408"  ; Request Time-out
                    | "409"  ; Conflict
                    | "410"  ; Gone
                    | "411"  ; Length Required
                    | "412"  ; Precondition Failed
                    | "413"  ; Request Entity Too Large
                    | "414"  ; Request-URI Too Large
                    | "415"  ; Unsupported Media Type
                    | "416"  ; Requested range not satisfiable
                    | "417"  ; Expectation Failed
                    | "500"  ; Internal Server Error
                    | "501"  ; Not Implemented
                    | "502"  ; Bad Gateway
                    | "503"  ; Service Unavailable
                    | "504"  ; Gateway Time-out
                    | "505"  ; HTTP Version not supported
                    | extension-code

    extension-code = 3DIGIT
    Reason-Phrase  = *<TEXT, excluding CR, LF>

    注: 客户不需要检查或显示Reason-Phrase。状态码的第一个数字定义了响应的类别，后两位数字没有任何分类角色。
        - 1xx: 信息   - 收到请求，继续处理
        - 2xx: 成功
        - 3xx: 重定向 - 必须采取进一步行动才能完成请求
        - 4xx: 客户端错误
        - 5xx: 服务器错误

    response-header  = Accept-Ranges
                     | Age
                     | ETag
                     | Location
                     | Proxy-Authenticate
                     | Retry-After
                     | Server
                     | Vary
                     | WWW-Authenticate
    general-header 和 response-header 之外的头字段为 entity-header。

```

**Entity**

如果没有受到请求方法或响应状态代码的限制，请求和响应消息可以传送实体。一个实体由entity-header和entity-body组成，尽管有些响应只包含entity-header。

```
    entity-header   = Allow
                    | Content-Encoding
                    | Content-Language
                    | Content-Length
                    | Content-Location
                    | Content-MD5
                    | Content-Range
                    | Content-Type
                    | Expires
                    | Last-Modified
                    | extension-header

    extension-header = message-header

    entity-body    = *OCTET
```

当消息包含entity-body时，该主体的数据类型通过头部字段Content-Type和Content-Encoding来确定。Content-Type指定底层数据的媒体类型。Content-Encoding可以用于指示应用于数据的任何附加编码，通常用于数据压缩的目的。任何包含entity-body的HTTP/1.1消息都应该包含一个定义该主体媒体类型的Content-Type头字段。当且仅当媒体类型不是由Content-Type字段给出时，接收者可以尝试通过检查媒体类型的内容和/或用于标识资源的URI的名称扩展来猜测媒体类型。如果媒体类型不明，接收者应该将其视为“application/octet-stream”。

```
    entity-body := Content-Encoding(Content-Type(data))
```

<h4>Connections</h4>

一个HTTP/1.1服务器可以假定一个HTTP/1.1客户端希望维持一个持久连接，除非在请求中发送了包含connection-token为“close”的连接头。如果服务器在发送响应之后选择立即关闭连接，它应该发送一个包含connection-token为“close”的连接头。

一个HTTP/1.1客户端可能希望连接保持打开状态，但会根据来自服务器的响应是否包含connection-token为“close”的连接头来决定保持打开状态。如果客户端不希望维持超过该请求的连接，它应该发送包含connection-token为“close”的连接头。

<h4><a href="https://tools.ietf.org/html/rfc2616#section-9">Method Definitions</a></h4>

**Safe and Idempotent Methods(安全方法和幂等方法)**

`GET`和`HEAD`被认为是安全的方法，而`POST`、`PUT`和`DELETE`则被认为是unsafe的。

`GET`、`HEAD`、`PUT`、`DELETE`、`OPTIONS`、`TRACE`是幂等的方法。

**OPTIONS**

OPTIONS方法代表请求关于在请求/响应链上可用的通信选项的信息，该请求/响应链由Request-URI标识。此方法的响应不可缓存。

**GET**

GET方法意味着检索任何由Request-URI标识的信息（以实体的形式）。如果Request-URI指向的是数据生成函数，则生成的数据将作为响应消息的实体返回，而不是数据生成函数。

如果请求消息包含`If-Modified-Since`，`If-Unmodified-Since`，`If-Match`，`If-None-Match`或`If-Range`头字段，则GET方法的语义变为“条件GET”。条件GET方法请求仅在条件头字段描述的情况下传送实体。条件GET方法旨在通过允许缓存实体刷新而不需要多个请求或传送客户端已经拥有的数据来减少不必要的网络使用。

如果请求消息包含`Range`头字段，则GET方法的语义变为“部分GET”。部分GET请求只传输实体的一部分，旨在通过允许完成部分检索的实体而不传送已经由客户端持有的数据来减少不必要的网络使用。

**HEAD**

HEAD方法与GET相同，只是服务器不能在响应中返回消息体。响应HEAD请求的HTTP头字段中的的元信息应该与响应GET请求而发送的信息相同。此方法通常用于测试超文本链接的有效性，可访问性和最近的修改。HEAD请求的响应可以是可缓存的，这意味着响应中包含的信息可以被用来更新一个先前被缓存的实体。

**POST**

POST方法用于请求源服务器接受请求中包含的实体作为Request-Line中Request-URI标识的资源的新下属(subordinate)。POST被设计为允许统一的方法来覆盖以下功能:

- 现有资源的注释;
- 向公告栏，新闻组，邮件列表或类似的文章组发布信息;
- 向数据处理过程提供一组数据，例如提交表单的结果;
- 通过追加操作扩展数据库

POST方法执行的实际功能由服务器决定，通常取决于Request-URI。

POST方法执行的操作可能不会创建可以通过URI标识的资源，这种情况下，根据响应是否包含描述结果的实体，200（OK）或204（无内容）是适当的响应状态。

如果操作创建了一个资源，那么响应应该是201（创建），并且包含一个描述请求状态的实体，并引用新的资源，一个Location头。

除非响应包含适当的Cache-Control或Expires头字段，否则对此方法的响应不可缓存。但是，303（See Other）响应可用于指示用户代理检索可缓存资源。

**PUT**

PUT方法请求将实体存储在响应的Request-URI下。如果Request-URI引用一个已经存在的资源，请求的实体应该被认为是驻留在源服务器上的修改版本。如果Request-URI指向的资源不存在，并且该URI能够被定义为新资源，则源服务器可以使用该URI创建资源。

**DELETE**

DELETE方法请求源服务器删除由Request-URI标识的资源。如果响应包括描述状态的实体，则成功的响应应该是200；如果该操作尚未实施，则响应202（接受）；如果该操作已经被实施但响应不包含实体，则响应状态应该是204。

**TRACE**

TRACE方法用于调用请求消息的远程应用层回环。请求的最终接收者应该将接收到的消息反映为200（OK）响应的实体主体。最终接收者是请求中的源服务器或第一个代理或接收到Max-Forwards值为0的网关。TRACE请求不能包含实体。

TRACE允许客户端查看请求链另一端正在接收的内容，并将该数据用于测试或诊断信息。


<h4><a href="https://tools.ietf.org/html/rfc2616#section-10">Status Code Definitions</a></h4>

状态码的含义应该被正确使用，我们都知道这非常重要。本文不逐一列举各状态码的含义及使用场景，如上标题链接到了原文的相应位置方便查看。

<h4><a href="https://tools.ietf.org/html/rfc2616#section-14">Header Field Definitions</a></h4>

**Accept**

`Accept` request-header字段可用于指定可接受的响应媒体类型。可以使用Accept头来表示请求被特定地限制在一小部分类型中，例如请求一个图像。

```
    Accept         = "Accept" ":"
                    #( media-range [ accept-params ] )

    media-range    = ( "*/*"
                    | ( type "/" "*" )
                    | ( type "/" subtype )
                    ) *( ";" parameter )
    accept-params  = ";" "q" "=" qvalue *( accept-extension )
    accept-extension = ";" token [ "=" ( token | quoted-string ) ]
```

**Accept-Charset**

`Accept-Charset` request-header字段用于指示可接受的字符集。如果没有Accept-Charset标头，则默认值是任何字符集均可接受。

```
    Accept-Charset = "Accept-Charset" ":"
                    1#( ( charset | "*" )[ ";" "q" "=" qvalue ] )

eg:
    Accept-Charset: iso-8859-5, unicode-1-1;q=0.8
```

**Accept-Encoding**

`Accept-Encoding` request-header字段与`Accept`类似，但限制了响应的内容编码。

```
    Accept-Encoding  = "Accept-Encoding" ":"
                       1#( codings [ ";" "q" "=" qvalue ] )

    codings          = ( content-coding | "*" )

eg:
    Accept-Encoding: compress, gzip
    Accept-Encoding:
    Accept-Encoding: *
    Accept-Encoding: compress;q=0.5, gzip;q=1.0
    Accept-Encoding: gzip;q=1.0, identity; q=0.5, *;q=0
```

**Accept-Language**

`Accept-Language` request-header字段与`Accept`类似，但限制了响应首选的自然语言集合。

```
    Accept-Language = "Accept-Language" ":"
                     1#( language-range [ ";" "q" "=" qvalue ] )

    language-range  = ( ( 1*8ALPHA *( "-" 1*8ALPHA ) ) | "*" )

eg:
    Accept-Language: da, en-gb;q=0.8, en;q=0.7
```

**Age**

`Age` response-header字段用来表示自响应生成依赖的估计时间。如果cache返回的响应其`Age`值没有超过 freshness lifetime，则视该响应为新鲜的。

```
    Age = "Age" ":" age-value
    age-value = delta-seconds
```

含缓存的HTTP/1.1服务器必须在从自己缓存中生成的响应中包含Age响应头字段。

**Allow**

`Allow` entity-header字段列出由请求URI标识的资源支持的一组方法。这个字段的目的是严格的通知接收方与资源有关的有效方法。`Allow` 必须出现在405（Method Not Allowed）响应中。

```
    Allow   = "Allow" ":" #Method

eg:
    Allow: GET, HEAD, PUT
```

**Authorization**

希望通过服务器验证自己的用户代理 —— 通常（但不一定）在收到401响应之后 —— 通过在请求中包含`Authorization` request-header字段来实现。

```
    Authorization  = "Authorization" ":" credentials
```

HTTP 访问认证见 [RFC2617: HTTP Authentication: Basic and Digest Access Authentication](https://tools.ietf.org/html/rfc2617)

**Cache-Control**

`Cache-Control` general-header字段用于指定请求/响应链上的所有缓存机制必须遵守的指令，这些指令通常覆盖默认的缓存算法。 Cache指令是单向的，因为在请求中存在指令并不意味着在响应中给出相同的指令。

```
    Cache-Control   = "Cache-Control" ":" 1#cache-directive

    cache-directive = cache-request-directive
                    | cache-response-directive

    cache-request-directive = "no-cache"
                            | "no-store"
                            | "max-age" "=" delta-seconds
                            | "max-stale" [ "=" delta-seconds ]
                            | "min-fresh" "=" delta-seconds
                            | "no-transform"
                            | "only-if-cached"
                            | cache-extension

    cache-response-directive = "public"
                             | "private" [ "=" <"> 1#field-name <"> ]
                             | "no-cache" [ "=" <"> 1#field-name <"> ]
                             | "no-store"
                             | "no-transform"
                             | "must-revalidate"
                             | "proxy-revalidate"
                             | "max-age" "=" delta-seconds
                             | "s-maxage" "=" delta-seconds
                             | cache-extension

    cache-extension = token [ "=" ( token | quoted-string ) ]
```

Cache-Control 指令可以分为以下几类:

- 限制什么是可缓存的         ;这些只能由源服务器执行
- 限制缓存可能存储的内容      ;这些可能由源服务器或用户代理执行
- 基本到期机制的修改         ;这些可能由原始服务器或用户代理执行
- 控制缓存重新验证和重新加载   ;这些只能由用户代理执行
- 控制实体的转换
- 缓存系统扩展

HTTP cache 这块的内容目前还没有掌握，需要结合实践多阅读几遍。

**Content-Encoding**

`Content-Encoding` entity-header字段用作媒体类型的修饰符。当出现时，它的值表示在实体体上应用了附加的内容编码，因此必须应用解码机制来获得content-type头字段引用的媒体类型。`Content-Encoding`主要用于允许文档压缩，而不丢失其底层媒体类型。

```
    Content-Encoding  = "Content-Encoding" ":" 1#content-coding

eg:
    Content-Encoding: gzip
```

**Content-Language**

`Content-Language` entity-header字段描述了实体的预期用户的自然语言。请注意，这可能与实体主体中使用的所有语言不相等。

```
    Content-Language  = "Content-Language" ":" 1#language-tag

eg:
    Content-Language: da
```

**Content-Length**

`Content-Length` entity-header字段表示entity-body的大小（八位个数的十进制表示）。

```
    Content-Length    = "Content-Length" ":" 1*DIGIT
```

**Content-Location**

当从请求资源的URI分离的位置访问该实体时，可以使用`Content-Location` entity-header字段为消息中的实体提供资源位置。

```
    Content-Location = "Content-Location" ":"
                      ( absoluteURI | relativeURI )
```

**Content-MD5**

[RFC 1864](https://tools.ietf.org/html/rfc1864) 中定义的`Content-MD5` entity-header字段是实体主体的MD5摘要，用于提供实体主体的端到端消息完整性检查（MIC）。

```
    Content-MD5   = "Content-MD5" ":" md5-digest
    md5-digest   = <base64 of 128 bit MD5 digest as per RFC 1864>
```

**Content-Type**

`Content-Type` entity-header字段指示发送给接收方的实体主体的媒体类型。

```
    Content-Type   = "Content-Type" ":" media-type

eg:
    Content-Type: text/html; charset=ISO-8859-4
```

**Date**

`Date` general-header字段表示消息起源的日期和时间，具有与 [RFC 822](https://tools.ietf.org/html/rfc822) 中的原始日期相同的语义。

```
    Date  = "Date" ":" HTTP-date

eg:
    Date: Tue, 15 Nov 1994 08:12:31 GMT
```

**ETag**

`ETag` response-header字段为请求的变体提供实体标记的当前值。

```
    ETag = "ETag" ":" entity-tag

eg:
    ETag: "xyzzy"
    ETag: W/"xyzzy"
    ETag: ""
```

**Expect**

`Expect` request-header字段用于指示客户端需要特定的服务器行为。

```
    Expect       =  "Expect" ":" 1#expectation

    expectation  =  "100-continue" | expectation-extension
    expectation-extension =  token [ "=" ( token | quoted-string )
                             *expect-params ]
    expect-params =  ";" token [ "=" ( token | quoted-string ) ]
```

**Expires**

`Expires` entity-header字段给出了响应被认为陈旧的日期/时间。一个陈旧的缓存条目通常不会被缓存返回，除非它首先经过原始服务器的验证。`Expires`字段的存在并不意味着原始资源将在此期间或之后更改或停止存在。`Cache-Control` 字段的 max-age 指令会覆盖 `Expires` 字段。

```
    Expires = "Expires" ":" HTTP-date

eg:
    Expires: Thu, 01 Dec 1994 16:00:00 GMT
```

**Host**

`Host` request-header字段指定被请求的资源的Internet主机和端口号，从用户或引用资源所给出的原始URI中获得。客户端必须在所有HTTP/1.1请求消息中的包含`Host`字段。

```
    Host = "Host" ":" host [ ":" port ]

eg:
    GET /pub/WWW/ HTTP/1.1
    Host: www.w3.org
```

**If-Match**

`If-Match` request-header字段与方法一起使用以使其条件化。先前从资源获得过实体的客户端可以通过在`If-Match`字段中包含其关联的实体标记(ETag)的列表来验证这些实体中的一个。该特性的目的是允许高效地更新缓存的信息，使其具有最少的事务开销。用于更新资源的请求(例如PUT)可能包含一个`If-Match`头字段，以表明如果与If-Match值对应的实体(单个实体标记)不再是该资源的表示，则该请求方法不能应用。

```
    If-Match = "If-Match" ":" ( "*" | 1#entity-tag )

eg:
    If-Match: "xyzzy"
    If-Match: "xyzzy", "r2d2xxxx", "c3piozzzz"
    If-Match: *
```

**If-Modified-Since**

`If-Modified-Since` request-header字段用于将请求方法条件化: 如果请求的资源自该字段指定的时间以来没有修改，则不会从服务器返回实体；相反，将返回一个没有任何消息体304(未修改的)响应。

```
    If-Modified-Since = "If-Modified-Since" ":" HTTP-date

eg:
    If-Modified-Since: Sat, 29 Oct 1994 19:43:31 GMT
```

**If-None-Match**

与 `If-Match` 类似，如果请求中包含`If-None-Match`字段，并且任意实体标记与请求中该字段包含的实体标记匹配，则源服务器必须不执行该请求方法。除非该请求带有的`If-Modified-Since`字段中的时间与该资源最后修改的时间不匹配。

如果所有的实体标记都不匹配，则服务器必须像不存在`If-None-Match`字段一样处理该请求方法，并且同时忽略`If-Modified-Since`字段。

```
    If-None-Match = "If-None-Match" ":" ( "*" | 1#entity-tag )
```

**If-Unmodified-Since**

如果请求的资源自该字段指定的时间以来未被修改，那么服务器应该执行请求的操作，就像`If-Unmodified-Since`字段不存在一样不存在。如果请求的资源自该时间以来改变过，则服务器必须不能执行请求操作，并回复412(Precondition Failed)响应。

```
    If-Unmodified-Since = "If-Unmodified-Since" ":" HTTP-date
```

**Last-Modified**

`Last-Modified` entity-header字段表示源服务器认为的该变量最后被修改的日期和时间。HTTP/1.1服务器应该尽量发送`Last-Modified`字段。

```
    Last-Modified  = "Last-Modified" ":" HTTP-date
```

**Location**

`Location` response-header字段用于将接受者重定向到请求uri以外的位置，以完成请求或标识新资源。对于201响应，位置是由请求创建的新资源的位置。对于3xx响应，位置应该指示服务器的首选URI，以便自动重定向到资源。

```
    Location       = "Location" ":" absoluteURI

eg:
    Location: http://www.w3.org/pub/WWW/People.html
```

**Referer**

`Referer` request-header字段允许客户端为服务器指定获取请求URI的资源的地址，该字段允许服务器生成用于兴趣、日志记录、优化缓存等资源的反向链接列表。它还允许跟踪用于维护过时的或错误的链接。如果Request-URI从没有它自己的URI(比如用户键盘输入)的源中获得，则不能发送Referer字段。

```
    Referer        = "Referer" ":" ( absoluteURI | relativeURI )

eg:
    Referer: http://www.w3.org/hypertext/DataSources/Overview.html
```

**Retry-After**

`Retry-After` response-header字段可用于503(服务不可用)响应，以指示请求客户端无法访问该服务的时间。该字段也可用于任何3xx(重定向)响应，以指示用户代理在发出重定向请求前等待的最短时间。

```
    Retry-After  = "Retry-After" ":" ( HTTP-date | delta-seconds )

eg:
    Retry-After: Fri, 31 Dec 1999 23:59:59 GMT
    Retry-After: 120
```

**Server**

`Server` response-header字段包含有关源服务器用于处理请求的软件的信息。

```
    Server         = "Server" ":" 1*( product | comment )

eg:
    Server: CERN/3.0 libwww/2.17
```

**Transfer-Encoding**

`Transfer-Encoding` general-header字段指示在消息主体上应用了哪些(如果有的话)转换类型，以便安全地将其传输到发送方和接收方之间。这与内容编码(content-coding)不同，因为转换编码是消息(message)的属性，而非实体(entity)。

```
    Transfer-Encoding       = "Transfer-Encoding" ":" 1#transfer-coding

eg:
    Transfer-Encoding: chunked
```

**User-Agent**

`User-Agent` request-header字段包含有关发出请求的用户代理的信息。

```
     User-Agent     = "User-Agent" ":" 1*( product | comment )
```

**Vary**

`Vary` 字段指示了决定是否允许缓存在不重新验证的情况下使用响应（响应是新鲜的）来回复后续请求的请求头字段集合。

```
    Vary  = "Vary" ":" ( "*" | 1#field-name )
```

**WWW-Authenticate**

`WWW-Authenticate` response-header字段必须包含在401(未经授权的)响应消息中。

```
    WWW-Authenticate  = "WWW-Authenticate" ":" 1#challenge
```

<h4><a href="https://tools.ietf.org/html/rfc2616#section-15">Security Considerations</a></h4>

POST与GET的区别。记得之前在哪看到过这样一个面试题，有人专门为此写了一篇博客，提到了面试者普遍的两个错误认知:

1. URI长度有限制，因此有些请求只能用POST，否则会导致GET请求因URI过长不被支持
2. GET方法不安全

看了HTTP的文档，里边确实说道了这两点，我反而觉得上面两种说法并不是错误的。该文档确实提到了有些客户端的实现限制了URI的长度，只不过这不是HTTP的限制。而GET方法不安全该文档在[Encoding Sensitive Information in URI's](https://tools.ietf.org/html/rfc2616#section-15.1.3)一节也提到了，原文如下:

```
    Authors of services which use the HTTP protocol SHOULD NOT use GET based forms for
    the submission of sensitive data, because this will cause this data to be encoded
    in the Request-URI. Many existing servers, proxies, and user agents will log the
    request URI in some place where it might be visible to third parties. Servers can
    use POST-based form submission instead
```

---
layout: post
title:  "根据IP查询国家及运营商【python 实现】"
date:   2016-08-03 10:47:53 -0400
categories: others
tags:
- python
- ipinfo
---

给定一个IP，我们可以从一些网站得到该IP的一些信息，如国家、运营商、地理位置等。例如[ipinfo][ipinfo]、[ipinfodb][ipinfodb]。

**IPInfo**
最开始我们查询的IP数量并不多，使用ipinfo就能满足需求，使用http连接与网站交互，获取地址信息：

> import netaddr
> import httplib
> 
> def is_internal_ip(ip):
> 　ipdec = int(netaddr.IPAddress(ip))
> 　ip1 = int(netaddr.IPAddress('127.0.0.1'))
> 　ip2 = int(netaddr.IPAddress('10.0.0.0'))
> 　ip3 = int(netaddr.IPAddress('172.16.0.0'))
> 　ip4 = int(netaddr.IPAddress('192.168.0.0'))
> 　if (ipdec >> 24) == (ip1 >> 24) or \
> 　　　(ipdec >> 24) == (ip2 >> 24) or \
> 　　　(ipdec >> 20) == (ip3 >> 20) or \
> 　　　(ipdec >> 16) == (ip4 >> 16):
> 　　return True
> 　return False
> 
> def get_country_and_ISP(ip):
> 　try:
> 　　if (is_internal_ip(ip)):
> 　　　raise Exception("%s is an internal ip" % ip)
> 　　else:
> 　　　url = 'http://ipinfo.io/' + ip
> 　　　cli = httplib.HTTPConnection('ipinfo.io', 80, timeout = 30)
> 　　　cli.request('GET', url)
> 　　　res = cli.getresponse()
> 　　　info = json.loads(res.read())
> 　　　if 'country' in info.keys() and 'org' in info.keys():
> 　　　　return info['country'], info['org']
> 　　　if 'country' in info.keys():
> 　　　　return info['country'], None
> 　　　if 'org' in info.keys():
> 　　　　return None, info['org']
> 　　　return None, None
> 　except Exception, e:
> 　　print ("get_country_and_ISP ip: %s error: %s" % (ip, e))

[ipinfo][ipinfo.io]的查询效率很高，速度很快，它一个缺陷是免费用户每天只能查询1000次，虽然我会对同一个IP的信息进行记录，但是我们每天查询的IP地址极有可能超过1000次，因此在不想掏钱的前提下不得不放弃使用它。

----------

**IPInfoDB**
然后改为IPInfoDB，它并不限制查询次数，但需要注册并获取一个token，使用这个token来进行相应的查询，但这个库似乎并没有提供运营商的信息，使用其提供的python库来获取国家和城市：

> import pyipinfodb
> 
> ip_lookup = pyipinfodb.IPInfo('your_token')
> 
> def get_country_and_city(ip):
> 　try:
> 　　if (is_internal_ip(ip)):
> 　　　raise Exception("%s is an internal ip" % ip)
> 　　else:
> 　　　ipinfo = ip_lookup.get_city(ip)
> 　　　if 'countryCody' in ipinfo.keys() and 'cityName' in ipinfo.keys():
> 　　　　return ipinfo['countryCode'], ipinfo['cityName']
> 　　　if 'countryCode' in ipinfo.keys():
> 　　　　return ipinfo['countryCode'], None
> 　　　if 'cityName' in ipinfo.keys():
> 　　　　return None, ipinfo['cityName']
> 　　　return None, None
> 　except Exception, e:
> 　　print ("get_country_and_city ip: %s error: %s" % (ip, e))

pyipinfodb的查询非常慢，因此需要把查询过的IP数据记录下来，下次遇到同一个IP直接从字典里取结果或大大提高效率。但随着需求的变更，我们需要获取IP所在的运营商，因此这个库不再能满足我们的需求。

----------

**IPWhois**
whois是一个用来查询域名是否已经被注册，以及注册域名的详细信息的数据库（如域名所有人、域名注册商）。[IPWhois](https://github.com/secynic/ipwhois)是一个通过IPv4或IPv6地址来获取并解析whois数据的python库，可以获取IP的国家和运营商的ASN(Autonomous System Number)，使用方法如下：

> from ipwhois import IPWhois
> 
> def get_country_and_asn(ip):
> 　try:
> 　　if (is_internal_ip(ip)):
> 　　　raise Exception("%s is an internal ip" % ip)
> 　　else:
> 　　　obj = IPWhois(ip)
> 　　　results = obj.lookup_rdap(depth=1)
> 　　　return results['asn_country_code'], results['asn']
> 　except Exception, e:
> 　　print ("get_country_and_asn ip: %s error: %s" % (ip, e))

使用ipwhois可以获取国家和asn，asn只是一个数字，并没有对应的名字，辨识度不好，因此我们需要把运营商的名字通过asn查询出来，使用[moocher.io](http://moocher.io/)提供的API进行查询：

> def get_ISP_by_asn(asn):
> 　try:
> 　　url = 'api.moocher.io/as/num/' + asn
> 　　cli = httplib.HTTPConnection('api.moocher.io', 80, timeout = 30)
> 　　cli.request('GET', url)
> 　　res = cli.getresponse()
> 　　info = json.loads(res.read())
> 　　return info['name']
> 　except Exception, e
> 　　print ("get_ISP_by_asn asn: %s error: %s" % (asn, e))

至此，我们可以通过IP开查询准确的国家和运营商信息了。

[ipinfo]: ipinfo.io
[ipinfodb]: ipinfodb.com

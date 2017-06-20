---
layout: post
title:  Proxying nginx to AWS
---
When moving hosts (in this case from a dedicated server to AWS), even if your DNS TTL is low, it is some useful to proxy traffic from the old environment to the new to reduce downtime for any stragglers. nginx makes this fairly straightforward but there are a couple of important config options when SSL is involved.

My nginx server looked like this:

```
location / {
     proxy_pass https://d1234567890.cloudfront.net;
     proxy_ssl_protocols TLSv1.2;
     proxy_ssl_server_name on;
     proxy_ssl_name example.com;

     proxy_set_header  Host              $host;
     proxy_set_header  X-Real-IP         $remote_addr;
     proxy_set_header  X-Forwarded-For   $proxy_add_x_forwarded_for;
     proxy_set_header  X-Forwarded-Proto $scheme;
     proxy_pass_header Authorization;
}
```

The `proxy_pass` directive tells nginx where to forward the traffic to. However, as the upstream is at CloudFront, we need some further options to make this actually work. `proxy_ssl_protocols` forces the SSL negotiation to use TLS 1.2, as CloudFront does not support SSLv3 and we might as well use the latest version. `proxy_ssl_server_name` enables SNI (Server Name Indication) on the upstream connection which is required as CloudFront handles multiple servers and needs to know upfront which one to connect to. Finally we also need to set `proxy_ssl_name` to tell nginx which name to send for SNI - without this it will send the name specified in `proxy_pass` which will work in some cases but if CloudFront has a certificate installed on the distribution, without this you will get errors like the following:

```
SSL_do_handshake() failed (SSL: error:14077410:SSL routines:SSL23_GET_SERVER_HELLO:sslv3 alert handshake failure) while SSL handshaking to upstream
```

Finally the header lines ensure that the correct headers are forwarded on over the proxy so the origin can discover the correct hostname, source IP address, protocol and authentication details if it needs them.

With all this in place I was able to migrate an existing server to AWS with no downtime, as the DNS could be left to propagate while the proxy took care of traffic still going to the old server.


---
title: "Using relayd as a reverse proxy on OpenBSD"
draft: true
---

I host different services ([httpd(8)](https://man.openbsd.org/httpd.8) and [Dendrite](https://matrix-org.github.io/dendrite/) primarily) and domains (including this site) on a server from [OpenBSD Amsterdam](https://openbsd.amsterdam/), and I recently added [atproto PDS self-hosting](https://mutualaid.cloud) to the mix. [relayd(8)](https://man.openbsd.org/relayd.8) handles all of the requests to the server, terminates TLS when needed, and routes requests between clients and their ultimate destinations. I could use something more common for this like nginx, but I generally like OpenBSD's base system service daemons because the documentation is excellent and the services are always incredibly stable.

When it comes to relayd, I've always been significantly more confused by its configuration thany any of the other services that I use. Maybe this is owed to relayd's flexibility and wide range of functionality (IP and application layer logic, TLS termination, health-checks, load balancing, etc.), or the infrequency with which I change its configuration.

This post is a simple, concise explanation of the features of relayd that I use. The examples are a bit contrived, but will demonstrate how the different components of relayd's (and httpd's) configuration work together to:

* Redirect HTTP requests to HTTPS
* Redirect subdomains to the apex domain (e.g., `www.example.com` to `example.com`)
* Route requests for certain HTTP paths to different services
* Route requests for certain domain names to httpd or different services
* Add custom headers to HTTP responses

## Key terms

Most of these are summaries of definitions from the [relayd.conf(5)](https://man.openbsd.org/relayd.conf.5) man page. If you want to read about any of the key terms in more detail, it's worth checking out. This post will **not** cover redirections (layer 3 or IP-based forwarding), global configuration settings like logging, health-checks, load balancing, or non-HTTP protocols.

### Tables

Tables are lists of hosts (similar to [pf tables](https://www.openbsd.org/faq/pf/tables.html)) that are used for target selection and health-checks.

### Relays

Relays operate on layer 7 (the application layer) of the [OSI model](https://en.wikipedia.org/wiki/OSI_model) (which has issues, but it's a useful shorthand and a lot of people still learn it) and allow for more advanced functionality like TLS termination and redirection based on HTTP headers and paths. Relays allow you to use relayd as an HTTP reverse proxy, or do more advanced application layer request routing (acting as a more generic application layer gateway). The relay configuration defines a port to listen and accepts client connections on.

The `forward to` directive of a relay tells it which table to forward connections to.

### Protocols

The protocol of a relay tells it which protocol specification to use (protocols are separate blocks in the configuration file). A protocol specification can be one of three types: `dns`, `http`, or `tcp`. A protocol directive is essentially a set of rules that applies to traffic received by any relay that uses it.

### Filter rules

Filter rules are available in the context of protocols, and allow you to do things like filter or manipulate traffic based on HTTP paths and header values. When you use the `forward to` parameter in a filter rule inside of a protocol directive, **there must also be a corresponding `forward to` declaration in the relay that uses the protocol**.

Having multiple `forward to` declarations in a relay is one aspect of relayd's configuration that often confuses me. It's helpful to think of these as *possible paths* that a relay will forward a client to, with filter rules determining which path is taken (without filter rules, the first `forward to` directive with any healthy hosts will be used).

## Putting it all together

To understand how tables, relays, protocols, and filter rules work together, let's walk through a simple example. The example won't explicitly mention when or how to restart relayd (`rcctl restart relayd`) (or how to use [relayctl(8)](https://man.openbsd.org/relayctl.8)) but generally you should restart it after any configuration change.

### Prerequisites

This example assumes that you have a valid TLS certificate for `example.com` (with [SANs](https://en.wikipedia.org/wiki/Public_key_certificate#Subject_Alternative_Name_certificate) `www.example.com` and `example-service.example.com`), with the certificate and private key stored at `/etc/ssl/example.com.crt` and `/etc/ssl/private/example.com.key` respectively. You can obtain a certificate using OpenBSD's [acme-client(1)](https://man.openbsd.org/acme-client.1) or a tool like [acme.sh](https://github.com/acmesh-official/acme.sh).

It also assumes the following simple httpd configuration is saved at `/etc/httpd.conf` and the httpd service is running (`rcctl start httpd`).

```
server "www.example.com" {
    listen on 127.0.0.1 port 8080
    listen on 127.0.0.1 port 8443
    block return 301 "https://example.com$REQUEST_URI"
}

server "example.com" {
    listen on 127.0.0.1 port 8080
    block return 301 "https://example.com$REQUEST_URI"
}

server "example.com" {
    listen on 127.0.0.1 port 8443
    root "/htdocs/example.com"
}
```

This configures httpd to listen only on localhost (127.0.0.1). Note that there is no TLS configuration here. httpd will assume that traffic arriving on port 8443 was sent by the client over HTTPS and TLS-terminated by relayd, and traffic arriving on port 8080 was sent by the client over HTTP. The latter (HTTP) traffic will be redirected to HTTPS. The first `server` section additional redirects any request (HTTP or HTTPS) for the `www` subdomain to the apex domain (this is just a personal preference).

### Initial relayd configuration

Save the following as `/etc/relayd.conf` (update `vio0` to your external network interface name):

```
table <httpd> { 127.0.0.1 }

http protocol "plaintext" {
    pass forward to <httpd>
}

http protocol "encrypted" {
    tls { keypair example.com no tlsv1.2 }
    pass forward to <httpd>
}

relay "http" {
    listen on vio0 port 80
    protocol "plaintext"
    forward to <httpd> port 8080
}

relay "https" {
    listen on vio0 port 443 tls
    protocol "encrypted"
    forward to <httpd> port 8443
}
```

This configures relayd with one table, two protocols, and two relays. The protocols have one filter rule each, allowing requests and forwarding them to the `<httpd>` table which contains a single host. Note that the filter rules that forward to the `<httpd>` table (this name is arbitrary and does not have to be `<httpd>`) have matching `forward to` declarations in their corresponding relays (the relay names are also arbitrary and do not have to be `http` and `https`).

This is a very simple configuration that serves a single site over HTTPS. Now we can add tables and filter rules to support more complex request routing.

### Routing HTTP paths to services

Say you have a service listening on port 8000 of localhost. Maybe it's a Node.js or Python service that you'd like to put a TLS reverse proxy in front of, and serve at `https://example.com/example-service`. Add the following table:

```
table <example-service> { 127.0.0.1 }
```

Edit the `encrypted` protocol:

```
http protocol "encrypted" {
    tls { keypair example.com no tlsv1.2 }
    pass quick path "/example-service" forward to <example-service>
    pass forward to <httpd>
}
```

Without `quick` in the filter rule all requests will end up matching the final, default `pass` rule that forwards to the `<httpd>` table, so this is important to include.

Edit the `https` relay:

```
relay "https" {
    listen on vio0 port 443 tls
    protocol "encrypted"
    forward to <httpd> port 8443
    forward to <example-service> port 8000
}
```

Requests to `https://example.com/example-service` should now be forwarded to your service running on port 8000.

### Routing domains to services

You can also easily route a subdomain to a service running locally. This example assumes the service is listening on port 8001 of localhost.

Add a table:

```
table <example-service-subdomain> { 127.0.0.1 }
```

Edit the `encrypted` protocol:

```
http protocol "encrypted" {
    tls { keypair example.com no tlsv1.2 }
    pass request quick header "Host" value "example-service.example.com" forward to <example-service-subdomain>
    pass quick path "/example-service" forward to <example-service>
    pass forward to <httpd>
}
```

Note the inclusion of the `request` parameter on the filter. This ensures that the filter rule only matches headers on HTTP requests and not responses. This parameter is not necessary on the `path` filter rule because this parameter is only applicable with the `request` direction.

Update the `https` relay:

```
relay "https" {
    listen on vio0 port 443 tls
    protocol "encrypted"
    forward to <httpd> port 8443
    forward to <example-service> port 8000
    forward to <example-service-subdomain> port 8001
}
```

Requests to `https://example-service.example.com` should now be forwarded to your service running on port 8001.

### Adding response headers

You may want to add HTTP headers to responses. This can be done by editing the `encrypted` protocol:

```
http protocol "encrypted" {
    tls { keypair example.com no tlsv1.2 }
    match response header set "X-Frame-Options" value "deny"
    pass request quick header "Host" value "example-service.example.com" forward to <example-service-subdomain>
    pass quick path "/example-service" forward to <example-service>
    pass forward to <httpd>
}
```

This adds the commonly recommended `X-Frame-Options` header to all HTTP responses.

## Final notes

This was a really helpful post for me to research and write as I make some minor updates to my `relayd.conf` to forward requests to a [self-hosted atproto PDS](/posts/a-rough-sketch-of-at-protocol-and-pds-self-hosting/). Hopefully it's helpful for others as well. Look for a post in the near future about PDS self-hosting on OpenBSD!

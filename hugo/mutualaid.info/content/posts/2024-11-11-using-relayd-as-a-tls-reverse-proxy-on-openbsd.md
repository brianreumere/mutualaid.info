---
title: "Using relayd as a TLS reverse proxy on OpenBSD"
draft: true
---

I generally like OpenBSD's base system service daemons (some created by their team, some created elsewhere and just shipped with the base system). In both cases, the documentation is always excellent and the services are always incredibly stable.

The relayd service was first shipped with OpenBSD 4.1 in May of 2007 and was called hoststated at the time (it was renamed relayd in OpenBSD 4.3). 

Its configuration has confused me significantly more than any other service that I use, so I wanted to write a post partly for my own sake about how relayd works and how to configure it.

## Key terms

**Tables** are lists of hosts (similar to pf tables) that are used for target selection and health checks.

Redirections operate on layer 3 of the OSI model (which I have issues with, but it's what people—including me—learn and is a useful shorthand) and use pf rdr-to rules to rewrite the destination IP address and port of certain traffic.

**Relays** operate on layer 7 of the OSI model and allow for more advanced functionality like TLS termination and redirection based on HTTP headers and paths. Relays allow us to use relayd as an HTTP reverse proxy, or do more advanced application layer request routing (acting as a more generic application layer gateway). The relay configuration is what actually accepts a client connection.

The `forward to` directive of a relay tells it which table to forward connections to. The **`protocol`** of a relay tells it which protocol specification to use (protocols are separate blocks in the configuration file). A protocol specification can be one of three types: `dns`, `http`, or `tcp`. This post only covers `http`-type protocol specifications.

**Filter rules** are available in the context of protocols, and allow you to filter or manipulate traffic based on HTTP paths and header values. When you use the `forward to` parameter in a filter rule inside of a protocol, **there must also be a corresponding `forward to` declaration in the relay that uses the protocol**. This is one aspect of relayd's configuration that always confuses me when I don't work with it for a while. Having multiple `forward to` declarations in a relay is a bit confusing.

To understand how tables, relays, protocols, and filter rules interact with each other, here's a simple HTTP-only example:

```
to-do
```



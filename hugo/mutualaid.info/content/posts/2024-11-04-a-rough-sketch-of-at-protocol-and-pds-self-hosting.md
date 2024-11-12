---
title: "A rough sketch of AT Protocol and PDS self-hosting"
draft: false
---

I've been learning a bit about AT Protocol recently as a side effect of taking some initial steps toward self-hosting my data (on the same server that powers this website). This post is part brain dump of what I've learned about the protocol and part rough sketch of the current state of PDS migration and self-hosting. If all goes well, I'll write a follow-up post about self-hosting a PDS on OpenBSD.

## First things 🇵🇸

I'd like to preface this post by sharing that Bluesky has either marked as spam or banned the accounts of a bunch of people from Gaza recently (presumably for trying to raise money for themselves to escape the ongoing Israeli genocide and occupation of Palestine). Here are a couple of examples shared by [Molly Shah](https://bsky.app/profile/mommunism.bsky.social/post/3l7fcbbqwbt2x) and [Fatima Ayub](https://bsky.app/profile/thecynicist.bsky.social/post/3l7svewwuhs2w).

I find these moderation decisions profoundly disgusting and immoral to take toward people trying to escape a genocide, and would encourage you to check out Fatima Ayub's [compilation of fundraisers for people she has personally connected with in Gaza](https://bsky.app/profile/thecynicist.bsky.social/post/3l6astrlnmy2q) as well as [Connecting Humanity](https://connecting-humanity.org/), which facilitates eSIM donations to people in Gaza who need internet access. I hope [Bluesky's head of trust and safety](https://bsky.app/profile/aaron.bsky.team) changes course on these decisions.

## PDSs, DIDs, and identity 

{{< callout >}}Some of the example commands in the rest of this post require [curl](https://curl.se/) and [jq](https://jqlang.github.io/jq/).{{< /callout >}}

[AT Protocol](https://atproto.com/) (short for Authenticated Transfer Protocol, also called atproto) is the protocol that the [Bluesky](https://bsky.social) microblogging platform is built on. Ever since finding out that it's possible to [self-host](https://atproto.com/guides/self-hosting) your [Personal Data Server (PDS)](https://atproto.com/guides/glossary#pds-personal-data-server) I've been interested in trying it out. A PDS is essentially a set of repositories ([data repositories or data repos](https://atproto.com/guides/glossary#data-repo) in atproto terms) that hold data associated with users. Users are represented by [DIDs (decentralized IDs)](https://atproto.com/specs/did), which are a [W3 Recommendation](https://www.w3.org/TR/did-core/).

Two types of DIDs are currently supported by atproto: [`did:web`](https://w3c-ccg.github.io/did-method-web/) (tied to ownsership of a domain name) and [`did:plc`](https://github.com/did-method-plc/did-method-plc) (currently hosted by the Bluesky-owned [PLC directory](https://web.plc.directory/), something they've expressed the desire to transfer to the ownership of an organization like [ICANN](https://en.wikipedia.org/wiki/ICANN) in the future). Both types of DIDs have an associated DID document that, among other things, maps a DID to its known handles and current PDS address. Web DID documents are typically found at `/.well-known/did.json`. PLC DID documents are returned by the [`ResolveDid` endpoint](https://web.plc.directory/api/redoc#operation/ResolveDid) of the PLC Directory Server API.

### DID example

My current Bluesky handle is `brian.mutualaid.info`. To resolve this to my DID I can use the [XRPC API](https://docs.bsky.app/docs/api/at-protocol-xrpc-api) hosted by Bluesky (and which should eventually be hosted on my own PDS):

```sh
curl -s 'https://bsky.social/xrpc/com.atproto.identity.resolveHandle?handle=brian.mutualaid.info' | jq -r .did
```

This returns my DID, `did:plc:xid65lb2n2iyqnxundqrvnpw`. Now I can query the PLC directory and get the address of my PDS:

```sh
curl -s https://plc.directory/did:plc:xid65lb2n2iyqnxundqrvnpw | jq -r .service
```

This returns the following JSON:

```json
[
  {
    "id": "#atproto_pds",
    "type": "AtprotoPersonalDataServer",
    "serviceEndpoint": "https://enoki.us-east.host.bsky.network"
  }
]
```

This shows that my current PDS service endpoint is `https://enoki.us-east.host.bsky.network`. Once I migrate to a self-hosted PDS, this should be something like `https://pds.mutualaid.cloud`. It's worth noting that once you migrate off of a Bluesky-owned PDS, you **can not** migrate back. My understanding is that this is due to technical performance limitations with the `importRepo` operation on large PDSs, and should theoretically be fixed in the future (allowing you to migrate back **to** a PDS on `bsky.network`)

## Other components of the Atmosphere

While the above components (DIDs and PDSs) are probably the most directly relevant to PDS self-hosting, there are other components of the atproto network (they call it the [Atmosphere](https://atproto.com/guides/glossary#atmosphere)) that are helpful to understand as well.

### Firehoses and relays

A [firehose](https://docs.bsky.app/docs/advanced-guides/firehose) is a stream of events published by either a PDS or a [relay](https://atproto.com/guides/glossary#relay) (a relay aggregates events from many PDSs and publishes them as a single stream of data). To receive data from a firehose, a client opens a WebSocket connection to a firehose provider. Relays are a performance optimization and aren't strictly necessary in atproto; clients may receive events directly from PDSs. [There is a bit more on relays in the Bluesky documentation](https://docs.bsky.app/docs/advanced-guides/federation-architecture#relay). Relays, if they don't filter events, are fairly resource intensive to run. [Notes on Running a Full-Network atproto Relay (July 2024)](https://whtwnd.com/bnewbold.net/entries/Notes%20on%20Running%20a%20Full-Network%20atproto%20Relay%20(July%202024)) from [bryan newbold](https://bnewbold.net/) (a protocol engineer at Bluesky) has some interesting detail on system resources and rough cost required to run a full-network atproto relay (about 150 USD/month at the time of writing). Since his post is on WhiteWind, it makes a good segue into talking about collections and Lexicon schemas.

### Collections, Lexicon, and AppViews

[WhiteWind](https://whtwnd.com/) is a Markdown-based blog service that uses your existing atproto PDS (at Bluesky or elsewhere, e.g. if you self-host) to host your blog content. It also surfaces comments made in the WhiteWind web interface as posts on Bluesky, and any posts on Bluesky with a `whtwnd.com` blog post URL as replies/comments in the WhiteWind web interface. To understand how of this works, in addition to the atproto components I've mentioned already we'll need to understand [collections](https://atproto.com/guides/glossary#collection) and a little bit about [Lexicon](https://atproto.com/guides/glossary#lexicon).

To see all of the collections that exist on my PDS, I can use the `describeRepo` XRPD API endpoint:

```sh
curl -s 'https://bsky.social/xrpc/com.atproto.repo.describeRepo?repo=brian.mutualaid.info&collection=' | jq -r .collections
```

This currently returns the following JSON:

```json
[
  "app.bsky.actor.profile",
  "app.bsky.feed.like",
  "app.bsky.feed.post",
  "app.bsky.feed.repost",
  "app.bsky.graph.block",
  "app.bsky.graph.follow",
  "app.bsky.graph.listblock",
  "chat.bsky.actor.declaration",
  "com.whtwnd.blog.entry"
]
```

Note the `"com.whtwnd.blog.entry"` value, which didn't exist before I'd made [my first post on WhiteWind](https://whtwnd.com/brian.mutualaid.info/entries/Hi). To explore the records in that collection, we can use the `listRecords` endpoint:

```sh
curl -s 'https://bsky.social/xrpc/com.atproto.repo.listRecords?repo=brian.mutualaid.info&collection=com.whtwnd.blog.entry' | jq -r .
```

This returns the following JSON:

```json
{
  "records": [
    {
      "uri": "at://did:plc:xid65lb2n2iyqnxundqrvnpw/com.whtwnd.blog.entry/3la5v2sq4s42q",
      "cid": "bafyreietfd75mn5mfmvg2vaix4mlwqfdqsqg6mcle4wotigmppr5m26h5y",
      "value": {
        "$type": "com.whtwnd.blog.entry",
        "theme": "github-light",
        "title": "Hi",
        "content": "# Hello, world\n\nI'm just testing out WhiteWind as part of exploring atproto and PDS self-hosting.",
        "createdAt": "2024-11-04T23:36:38.150Z",
        "visibility": "url"
      }
    }
  ],
  "cursor": "3la5v2sq4s42q"
}
```

The `records` array contains all of the posts I've made on WhiteWind. I can similary find all of the posts I've made on Bluesky by changing the `collection` query parameter value to `app.bsky.feed.post`.

These collection names are [NSIDs](https://atproto.com/guides/glossary#nsid-namespaced-id) (or Namespace IDs), and are identifiers for Lexicon schemas (which are formatted as reverse DNS names). There is some technical background of why Bluesky developed Lexicon in the post [Why not RDF in the AT Protocol?](https://www.pfrazee.com/blog/why-not-rdf) by Paul Frazee. In short, each collection type (as well as XRPC API endpoints and event stream messages, but I'm not going to get into those here) is defined by a Lexicon schema (the [Lexicon spec is here](https://atproto.com/specs/lexicon)) and applications that use the atproto network can create arbitrary new collections with schemas that fit their intended use. [Schemas aren't necessarily available via an API](https://atproto.com/guides/lexicon#schema-distribution) but may be published somewhere. For example, [various Bluesky lexicons are here](https://github.com/bluesky-social/atproto/tree/main/lexicons).

I won't say too much about [AppViews](https://atproto.com/guides/glossary#app-view), but they're basically applications that make use of the atproto network (and all or most of the components of it that I've mentioned so far). An AppView might be a web app, a mobile app, or a CLI app; it's basically a particular view of data stored on the atproto network.

## Wrapping up

It's been fun learning a bit more about atproto (I've been meaning to for a while). There's one comment from [dan abramov](https://danabra.mov/) that [sums up some of the central components of atproto very succintly](https://news.ycombinator.com/item?id=41487673) that I thought it would make sense to close with.

I have the [reference PDS implemention](https://github.com/bluesky-social/atproto/tree/main/packages/pds) running on OpenBSD now, and just need to spend some time setting up things like storage and other required environment variables before migrating my PDS. I'm a *little* wary since I can't migrate back to a Bluesky-hosted PDS, so I'll probably at least test things out with a separate Bluesky account to start with. I'll hopefully have a follow-up post covering PDS migration up in the near future!

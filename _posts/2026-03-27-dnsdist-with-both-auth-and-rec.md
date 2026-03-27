---
layout: post
title:  "Dnsdist with both authoritative and recursive services"
tags: dns dnsdist powerdns
---

On one hand this post may be stating the obvious, but having seen the topic come up now and then and the "obvious" being overlooked I'll take a moment to outline what I feel is "the way" to do this...


### Question
If, for whatever reason, you have one dnsdist instance in front of both authoritative and recursive services (maybe a legacy design decision), how do you choose where to direct any given query?

### Answer
[`RDRule`](https://www.dnsdist.org/reference/selectors.html#lua-RDRule).

Yes, that's it. Just checking the `RD` flag of the query is normally all there is to it.

The clients are literally stating which type of service they expect in their queries. And they are not wrong about what service they expect.


#### Example config snippet

```lua
addAction(NotRule(RDRule()), PoolAction('auth'))

addAction(AllRule(), PoolAction('rec'))
```

#### Rationale
No need for dnsdist to know which zones exist at auth, in fact deciding based on such a list instead of the `RD` flag does not only add configuration complexity, it actually causes real problems.

Example: A client expecting recursion gets directed directly to the authoritative server. The authoritative server encounters a `CNAME` but behaves like an authoritative server and does not follow the `CNAME`. Resolution breaks for the client.

In fact, `RD` generally just works. If you have clients trying to look up names whether on the Internet at large or any local zones, they set `RD` and get the recursor as they expect. If you have clients that are recursors, they don't set `RD` and end up at the authoritative server. If you have clients sending RFC2136 DNS updates, they don't set `RD` and they end up in the right place as well.


Now of course, the recursor that was part of this question does need to be able to query the zones on the authoritative server. If these zones are normal delegations in the broader DNS tree, it should just work. If on the other hand these are not proper delegations but zones that only exist in this local environment (quite possibly part of the point of why a "mixed use" server like this was created), then the recursor will probably need some kind of forward zone configuration, eg the PowerDNS recursor [forward zone](https://doc.powerdns.com/recursor/yamlsettings.html#forward-zone).


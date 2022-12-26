---
layout: post
title:  "PowerDNS Recursor RPZ with cacheable results"
tags: dns powerdns powerdns-recursor rpz
---

RPZ is a standardized and fairly convenient way of distributing and applying blocklists in standard nameserver software.
For instance, you may have lists with domains used for malware, tracking, ads, etc, to block, much like specialized
nameserver software such as PiHole does.  

Blocking names in DNS is not unproblematic, it is not a fool-proof way of blocking badness, it does not play nice
with DNSSEC (however, the goal is to block, so even triggering a validation failure achieves the end goal), and can be used in abusive ways.  
All that said, it can still certainly serve a purpose as a relatively unintrusive yet convenient layer of protection, and as long as it is used as a means of empowering the user, I at least don't see an ethical problem.

Setting up [RPZ in PowerDNS Recursor](https://doc.powerdns.com/recursor/lua-config/rpz.html#configuring-rpz) is pretty straightforward through its Lua-based configuration. See the below example:

recursor.conf:
```
lua-config-file=/etc/powerdns/recursor.lua
lua-dns-script=/etc/powerdns/dns.lua
extended-resolution-errors=yes
```

recursor.lua:
```lua
local rpz_zones={ "adblocknocoin.hoshsadiq.rpz.qw.se", "adguard.firebog.rpz.qw.se",
"adservers.anudeepnd.rpz.qw.se", "easyprivacy.firebog.rpz.qw.se", "hole.certpl.rpz.qw.se",
"hosts.stevenblack.rpz.qw.se", "kadhosts.stevenblack.rpz.qw.se", "notrack-blocklist.quidsup.rpz.qw.se",
"pgl.yoyo.rpz.qw.se", "prigent-malware.firebog.rpz.qw.se", "rpilist-malware.firebog.rpz.qw.se",
"rpilist-phishing.firebog.rpz.qw.se", "rpz.oisd.nl.rpz.qw.se", "simplead.disconnectme.rpz.qw.se",
"simpletracking.disconnectme.rpz.qw.se", "urlhaus.abusech.rpz.qw.se" }

for i,v in ipairs(rpz_zones) do
    rpzPrimary({ "139.162.158.28", "2a01:7e01::f03c:91ff:fee4:c68b" }, v, { extendedErrorCode = 15,
extendedErrorExtra = "Blocked by policy " .. v, dumpFile = "/tmp/db." .. v, seedFile = "/tmp/db." .. v,
axfrTimeout=60 })
end
```

This defines a list of RPZ zones to `AXFR`/`IXFR` from a server hosting some such zones, also defining an
extended error code to include in responses for blocked names.

What you will find, however, is that when PowerDNS Recursor applies an RPZ policy, it responds with empty `NXDOMAIN` messages (no `SOA`), while negative caching is done precisely based on the `SOA` record supposedly found in the `AUTHORITY` section of negative responses.  
This makes for uncacheable responses (or at least with no control at all over any caching behavior that may exist), however, one can use the [Lua hooks for changing responses](https://doc.powerdns.com/recursor/lua-scripting/hooks.html) to add a made-up `SOA` record that at least allows for specifying a TTL to control cache behavior.

dns.lua:
```lua
function nxdomain(dq)
    return fixrpz(dq)
end
function nodata(dq)
    return fixrpz(dq)
end

function fixrpz(dq)
    if dq.appliedPolicy.policyKind == pdns.policykinds.NODATA or dq.appliedPolicy.policyKind ==
pdns.policykinds.NXDOMAIN then
        pdnslog("Intercepting RPZ NODATA/NXDOMAIN for: "..dq.qname:toString())
        if dq.appliedPolicy.policyKind == pdns.policykinds.NODATA then
            dq.rcode = pdns.NOERROR
        else
            dq.rcode = pdns.NXDOMAIN
        end
        dq:addRecord(pdns.SOA, "blocked.invalid. blocked.invalid. 1 1 1 1 600", 2, 600)
        -- 2 = AUTHORITY - https://docs.powerdns.com/recursor/lua-scripting/dnsrecord.html#DNSRecord.place

        dq.appliedPolicy.policyKind = pdns.policykinds.NoAction
        return true
    end
    return false
end
```

This overrides `NXDOMAIN`/`NODATA` responses that were introduced by RPZ so that they include an `SOA` record.

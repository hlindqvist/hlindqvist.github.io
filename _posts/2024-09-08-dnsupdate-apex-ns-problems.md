---
layout: post
title:  "RFC2136 DNS Update Problems Caused by Non-atomic Handling of Apex NS RRSet"
tags: dns rfc2136 dnsupdate
---

[RFC2136](https://www.rfc-editor.org/rfc/rfc2136.html) DNS Updates provide an overall nice standardized interface for editing DNS zone contents over the network.

However, as [I encountered a while back](https://gitlab.isc.org/isc-projects/bind9/-/issues/4305) the spec mandates some special-case handling of the apex `NS` RRSet, which unfortunately makes for some very confusing behavior when using the UPDATE mechanism for generic zone editing (and breaks the overall promise of atomicity).

Specifically, there are two parts of the spec which interact:

* The spec mandates that the server must silently skip any deletion of the last `NS` RR as it goes through the changes in the UPDATE message, even if the message adds new `NS` RRs later on. (This is the actual problematic requirement.)
* There is an asymmetry between additions and deletions, such that deletions cannot match by TTL.

The end result is an issue where, depending on whether you order your changes in the UPDATE message as add-then-remove or remove-then-add, you get one these two undesirable outcomes:

* A TTL-only change of the `NS` RRSet deletes all but one of the `NS` RRs (add-then-remove)
* Changing the RData of every `NS` RR leaves one of the old RDatas behind (remove-then-add)


After some different attempts, with some trial and error, I have now [settled on this hack](https://github.com/hlindqvist/dnstools/commit/7ea95b188038d0148aa7371a8bae2edea98cd8e4) for one of my projects:

Detect if the apex `NS` RRset is being both added to and deleted from (the problematic scenario). If so, start the UPDATE message with the addition of a nonsense `NS` RR and end it with the removal of that same nonsense `NS` RR.

With this extra nonsense `NS` RR present at the start of the UPDATE message, the problematic silent server-side "skip removal of last `NS` RR" behavior can no longer trigger (the new nonsense `NS` is still there while removing other `NS` RRs), allowing a normal remove-then-add strategy for actual useful RRs to function also for the apex `NS` RRSet, and with the removal of the nonsense RR at the end of that same UPDATE message, the nonsense RR will be removed as part of the same transaction where it was introduced.


Is this really the way it's supposed to be done? Idk, I guess this is one of these cases where you realize that the spec has some problematic behaviour defined and you just have to come up with something that still works within those rules.

I think what the spec should have been is something like this:

If, after applying all the changes in the UPDATE message, there is no `NS` RR at the zone apex, it should refuse the UPDATE altogether (do not commit changes, return error status to client).
That way, it would only trigger if the UPDATE as a whole actually causes a problem, and it would actually allow for atomicity, not this "part of your update happened, other parts were discarded, return success" behavior that we run into now.

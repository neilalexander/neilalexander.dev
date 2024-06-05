---
layout: post
title: "On Canonical JSON in Matrix"
date: 2024-06-05 23:10:00 +0100
author: Neil Alexander
index: true
---

# On Canonical JSON in Matrix

Today, the Matrix protocol is pretty much entirely JSON over HTTP. This is true of the client-server protocol, the server-server (federation) protocol or any of the other supporting protocols (i.e. for appservices/bridges). When you send a message into a room, a persistent data unit (PDU, otherwise known as an “event”) is generated which is also JSON. Same for typing notifications, read receipts, encryption key exchanges, E2E device list updates. It’s all JSON, all the way down.

In order for Matrix to be able to operate as a largely decentralised federated system, cryptographic signatures are necessary in order for other servers in the federation to be able to verify that an event or a request truly originated from the server that claims to be the origin. These signatures form a critical part of the trust chain in the Matrix protocol and are important to prevent forgery and to prove authenticity, which means that we need a reproducible way to sign a JSON blob and know that other servers will be able to verify that signature correctly (or indeed to correctly classify a signature as invalid due to tampering). Not only are events signed but so are all federation request bodies between different servers, for much the same reason. 

To add to the complexity, Matrix also defines special case keys that are *excluded* from being signed from certain objects (such as the aptly-named `”unsigned”` key, or indeed other inline `”signatures”`), so it is necessary not just for the origin server to manipulate the JSON to exclude certain keys when generating the signature, but also for receiving servers to be able to do the same, so that unsigned regions can be transmitted inline but also not be covered by the signature, which is included inline.

Encoded JSON can also include whitespace and padding which serves no particular purpose other than to please the human eye. Key ordering is not prescribed either — that’s down to the encoder — so different JSON implementations might order keys in the encoded output differently. 

Cryptographic verification knows nothing of the intricacies of JSON though — a sign or verify function just wants a stream of bytes, and those streams could produce different signatures if they have different key ordering, different amounts of whitespace in them or any differences in the encoded JSON output at all. Therefore the Matrix specification (v1.10 at the time of writing) describes a canonicalised JSON form as a means of avoiding these issues:

> We define this encoding for a value to be the shortest UTF-8 JSON encoding with dictionary keys lexicographically sorted by Unicode codepoint. Numbers in the JSON must be integers in the range `[-(2*53)+1, (2*53)-1]`, represented without exponents or decimal places, and negative zero -0 MUST NOT appear.

I have argued many times before that this definition is insufficient (and have been accused of “kvetching” for doing so) ultimately because it both overstretches by trying to solve more than it should, and yet falls short on the problem it is intended to solve, which is to produce a form suitable for signature generation and validation. 

Fundamentally, we only actually need to do four things in order to satisfy the signature case:

1. Remove keys which should not be covered by the signature;
2. Sort keys into a reproducible order, lexicographically, byte by byte;
3. Remove duplicate keys (which shouldn’t be there anyway) by always preserving only the first value for that key, so that all implementations keep the same value;
4. Strip whitespace characters (tabs, carriage returns, line feeds and or spaces), as doing any of the above would affect existing whitespace anyway.

That’s it — nothing else. If you pull the JSON bytes from the wire and perform these steps exactly and reproducibly, then you will be able to safely feed the canonicalised form into a signature verify function and get the same result as anyone else who implements the same canonicalisation algorithm, even if they disagree with you on what constitutes good value formatting or what constitutes semantically valid values at all. 

The signature validation doesn’t care remotely about whether or not the JSON values are sane or appropriate, it only cares that it can be canonicalised down into the same byte-for-byte representation that the origin used when signing it, and it is this that should kept in mind when deciding exactly what the canonical form should achieve. It is better not to conflate the logically separate concerns of signature validation and value validation.

Note that I proposed above that lexicographical sorting is done byte-by-byte and not by Unicode codepoints as the Matrix spec currently does. This is a deliberate decision as the canonicalisation algorithm still needs to be able to function correctly in the face of invalid UTF-8 codepoints — the goal should be to check the signature at this stage, nothing else. 

Any further modifications to the raw JSON, such as reinterpreting how a value is represented, modifying or dropping invalid values, or even trying to correct or drop invalid UTF-8 codepoints in either the keys or the values, are all guaranteed ways of unintentionally breaking the signature validation, which can have far-reaching secondary consequences. Therefore it is **never** safe to round-trip the JSON through a decoder and encoder in an attempt to canonicalise it, as some reinterpretation or reformatting could happen in the process. 

No, seriously, I mean it: you *cannot* trust your JSON library to do the right thing when round-tripping. Any potential place where two different implementations can disagree on whether a signature is valid or not is dangerous in Matrix. Those cases result in split-brain scenarios which can lead to different servers disagreeing on state for the same room, potentially sending broken state to other servers in response to `/state` or `/state_ids` requests, or worse, omitting what *should be* valid events when providing inputs to the state resolution algorithm leading to state resets that could reset memberships, reverse bans or other power changes on some servers. (This is not hypothetical. I have researched dozens of cases of signature validation disagreements between servers causing state resets whilst working on Dendrite.)

Even Synapse, the flagship implementation, makes this exact mistake: it round-trips the JSON through a decoder and encoder and relies on the `sort_keys` option in Python’s JSON library (and the fact that it doesn’t include extraneous whitespace by default) to implement canonicalisation, which means that it is not immune to these types of problems either, as it can potentially round-trip values into a different result or format. Once this happens, the signature is no longer valid. The Python project also do not document their encoding behaviours so it is practically impossible to faithfully reproduce them in another language, and this is assuming they don’t change anything on a whim in a future Python version without telling anyone. Following the Matrix spec alone when trying to implement a new homeserver actually doesn’t get you that far and not being able to accurately recreate all of Synapse’s mysterious undocumented behaviours is a very real road block when it comes to trying to write an interoperable implementation in another language. 

The only way to avoid this realistically is to specify and implement a canonicalisation algorithm in a similar way to how gomatrixserverlib currently does — one that walks the JSON by hand, removing whitespace, sorting keys and, most importantly, *leaving the contents of keys and values alone* in the process. One that is simple enough to be feasibly implemented in any language and, critically, one that can be completely and unambiguously specced in the Matrix specification.

Which brings us to another little secret that is definitely not mentioned in the Matrix spec: the method in which the JSON is pulled from the wire and is stored matters. It is a known and well-established problem that different JSON encoder and decoder implementations have their own quirks and biases and often these can manifest in the encoded output or even when decoding. 

A good example of this is a very large number like `1234567890123456789`. When encoding from an integer type, Go will encode this number as-is with no change to formatting. However, from a floating point type, Go will round this number to `1234567890123456800` in the encoded output (which is to be expected due to the limited floating point precision). Python 3.12 reformats the floating point form to `1.2345678901234568e+18` — different bytes on the wire. Serde in Rust encodes the floating point version differently again, to `1.2345678901234568e18`. All of this is concerning when you remember that JSON doesn’t actually distinguish different numeric types at all, before you even look at the possibly-strange coercive behaviour of some dynamically-typed languages.

You might point out that, restrictions on floats aside, numbers this large are also technically forbidden by the Matrix canonical JSON specification. You’d be right, except that this wasn’t enforced for a number of years for room versions 1 to 5, so it is quite likely that there are events out there in the real world with these very large numbers in them, in rooms that are still active and have never been upgraded. This isn’t the only footgun either, there are [many many more]( http://seriot.ch/json/parsing.html).

Furthermore, if we want to be able to send these events to other servers as normal so that these room versions continue to function across the federation, then we also **cannot** enforce these value restrictions at the top federation request level either when reusing the canonicalisation algorithm, as the entire JSON request body could potentially contain these old events. Those request bodies would *also* need to be canonicalised in their complete form so that they too can be cryptographically signed. 

To return to the point about pulling JSON from the wire, the above problem effectively means that parsing JSON in the Matrix federation protocol often needs to be a multi-stage process, for two reasons:

1. A minimal canonicalisation algorithm, such as the one I propose above, relies heavily on the fact that there has been no reinterpretation or reformatting of keys or values before the JSON is fed into the canonicalise function;
2. Different rules might apply to different sections of the JSON, for example, a `/_matrix/federation/v1/send` call which contains events for multiple different room versions within the same transaction, with different restrictions on the values.

All of this means that it is never safe to persist JSON blobs that can be repeated/retransmitted later (i.e. PDUs/events) in a round-tripped form. Ideally you’d store any JSON in the exact original format in which you pulled it off the wire.

Go has a rather elegant solution for this in that a custom decoder implementation always receives the bytes from the wire exactly and that the special `json.RawMessage` type can be used to defer the decoding of specific keys until later. Therefore the troublesome sections can be decoded and validated in later decoding passes and also allows the original JSON as it was presented on the wire to be accessed and stored exactly. Many other JSON libraries provide similar mechanisms.

Note that I also haven’t mentioned numeric ranges, decimal places or negative zeros in my proposed steps above, even though the current Matrix spec does. This is intentional, as this takes us back to separation of concerns. These are all things that have no bearing whatsoever on signature validation and therefore do not belong in the canonicalisation algorithm. In fact, any further post-processing or validation of the JSON should be done *after* the signature has been validated or rejected as, by that time, you would know whether to even trust the event or not. These “good value” vs “bad value” considerations belong elsewhere.

I have long advocated for Matrix to one day move to a binary encoding format for the federation protocol that has special signed and unsigned regions within a payload, such that canonicalisation acrobatics aren’t necessary at all. Instead, being able to straight-forwardly validate entire sections as they are. This would remove a whole class of bugs and undo a great deal of uncertainty around whether or not the JSON signing in Matrix is robust or not. However, there is no denying that this would involve a massive amount of work.

Instead, specifying and moving to a strict, reproducible and fit-for-purpose canonicalisation algorithm that is precise in its scope (that is, to only enable cryptographic signing and not to try to enforce value sanity) would go a long way towards solidifying the foundations of the protocol and strengthening its security and interoperability guarantees. It could even be done in a backwards-compatible way.

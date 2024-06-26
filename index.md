---
---

I live in the North East of England and I am a Software Developer working
on [NATS](https://nats.io) at [Synadia](https://synadia.com). I have a keen
interest in security, privacy, decentralised systems and computer networking.
I also have experience in end-user compute and enterprise-scale systems architecture.

{% assign indexable_posts = site.posts | where: "index",true %}

{% unless indexable_posts.size == 0 %}

## Latest

<ul>
{% for post in indexable_posts limit:3 %}
    <li>
      {{ post.date | date: "%d %b %Y" }} — <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
{% endfor %}
</ul>
{% endunless %}

## Projects

**yggdrasil** (Maintainer, [GitHub](https://github.com/yggdrasil-network/yggdrasil-go), [Website](https://yggdrasil-network.github.io))
— A cross-platform prototype of an end-to-end encrypted IPv6 overlay network,
written in Go. It implements a new and experimental compact routing scheme based
around a globally-agreed spanning tree.

**pinecone** (Author, [GitHub](https://github.com/matrix-org/pinecone))
— A prototype of a name-independent, self-healing compact routing scheme, designed
for mesh networks and peer-to-peer applications, written in Go. Originally created
to underpin the P2P Matrix project.

**dendrite** (Contributor, [GitHub](https://github.com/matrix-org/dendrite), [Website](https://matrix.org/docs/projects/server/dendrite))
— A next-generation Matrix homeserver, written in Go, designed to operate at scales
ranging from single-user embedded homeservers to large-scale polylith deployments.

**jnacl** (Author, [GitHub](https://github.com/neilalexander/jnacl))
— Pure Java implementation of some NaCl ECC encryption and authentication primitives,
including the Curve25519 elliptic curve, Xsalsa20 stream cipher and Poly1305
message authentication algorithm.

## Contact

You can contact me by email at the following address:

<img src='assets/images/email.png' width='280px' id='contactbyemail' />

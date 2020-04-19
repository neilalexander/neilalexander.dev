---
---

# jnacl

jnacl is a library that implements some of Daniel J. Bernstein's NaCl encryption
primitives. The functions are written in pure Java, and are therefore very
straight-forward to make use of in a Java application without the use of native
libraries and bindings.

The following primitives are included:

* `curve25519`: Elliptic curve key agreement
* `salsa20`: Stream cipher
* `poly1305`: Message authenticator (MAC)

It is open-source and [available on GitHub](https://github.com/neilalexander/jnacl).

Even though it has not been comprehensively benchmarked, jnacl performs
exceptionally well, even on resource-constrained Android platforms.

---
layout: page
title: chacha20-poly1305 encryption algorithm drafts
---

## Drafts

- `ssh-chacha20-poly1305@openssh`:

  The original protocol used by openssh (sadly the upstream
  documentation is broken)

- ssh-chacha20-poly1305

  An updated protocol using AEAD_CHACHA20_POLY1305 from RFC7539.

## Motivation

When I first started implementing "chacha20-poly1305@openssh.com" I was
greatly disappointed by the qualitify of the documentation.

Finding the draft "draft-josefsson-ssh-chacha20-poly1305-openssh-00" I
hoped for some improvements, but it sadly is just the same document as
present in openssh-portable.

I read some of the discussions following the draft announce, and found
the following design questions:

- Whether to use the Poly1305 data construction from RFC7539:

  At first I thought the Poly1305 usage in
  "chacha20-poly1305@openssh.com" is vulnerable to some length
  modifications; but then I remembered that Poly1305 uses an explicit
  termination in the padding.

  As the length of the AAD is fixed I see no security gain changing to
  the method described in RFC7539.

- Whether it is necesary to encrypt the packet length field:

  Some voiced a strong preference for this as a requirement to prevent
  traffic analysis.

  It was pointed out the encrypted could lead to some extra attack
  surface (or has done so in other protocols in the past), but I think
  it is safe in the context of Chacha20. I see nothing an attacker could
  gain here.

- Encrypting the packet length using otherwise discarded bytes from the
  Chacha20 block used for the Poly1305 key:

  It is a nice idea which can be used to optimize both performance and
  memory usage.

- Binary packet protocol rethink:

  This is certainly worth exploring, but I don't think this needs to be
  completed before "chacha20-poly1305" becomes official.

  I think a new family of algorithms could be started with a different
  binary packet protocol, or even a new SSH protocol version.

  (My idea for a new protocol: separate encryption framing and inner
  message framing. No need to hide the outer packet length anymore.)

- Changing padding requirements, authentication of the encrypted length:

  I see no need to change this in the context of a single algorithm.
  Belongs into a more generic protocol redesign.

Changing the binary packet protocol probably requires rewriting core
logic in many SSH implementations, so this should be done very carefully
and not just for one cipher.

Until this happens I propose defining "chacha20-poly1305" as either the
existing "chacha20-poly1305@openssh.com" or as a slightly modified
variant:
- using the RFC7539 Poly1305 data construction
- using the Chacha20 variant from RFC7539
- encrypt the packet length with otherwise discarded bytes from the
  first Chacha20 block, i.e. only a single Chacha20 instance
- pad the nonce to 12 bytes with zeroes on the left side, so one can
  simply reuse the original Poly1305 implementation with a 8-byte nonce.
- I do have an openssh patch for this :)

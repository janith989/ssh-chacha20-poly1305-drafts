%%%
title = "The chacha20-poly1305@openssh.com authenticated encryption cipher"
abbrev = "SSH chacha20-poly1305@openssh.com"
category = "info"
ipr="trust200902"
docName = "draft-josefsson-ssh-chacha20-poly1305-openssh-00"
area = "Security"
workgroup = "Secure Shell (Concluded WG)"
keyword = ["Internet-Draft"]

[[author]]
initials="D."
surname="Miller"
fullname="Damien Miller"
organization="OpenSSH"

[[author]]
initials="S."
surname="Josefsson"
fullname="Simon Josefsson"
organization="SJD AB"
[author.address]
email="simon@josefsson.org"

[[author]]
initials="S."
surname="Bühler"
fullname="Stefan Bühler"
organization="RUS-CERT"
[author.address]
email = "buehler@cert.uni-stuttgart.de"
%%%

.# Abstract

This document describes the `chacha20-poly1305@openssh.com`
authenticated encryption cipher supported by OpenSSH.

{mainmatter}

# Introduction

ChaCha20 is a stream cipher designed by Daniel Bernstein and described
in [@!ChaCha].  It operates by permuting 128 fixed bits, 128 or 256 bits
of key, a 64 bit nonce and a 64 bit counter into 64 bytes of output.
This output is used as a keystream, with any unused bytes simply
discarded.

Poly1305 [@!Poly1305], also by Daniel Bernstein, is a one-time
Carter-Wegman MAC that computes a 128 bit integrity tag given a message
and a single-use 256 bit secret key.

The "`chacha20-poly1305@openssh.com`" cipher combines these two
primitives into an authenticated encryption mode.  The construction used
is based on that used with TLS  in [@!RFC7539], but differs in the
layout of data passed to the Message Authentication Code (MAC) and in
the addition of encyption of the packet lengths.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in
[@!RFC2119].

# Negotiation

The `chacha20-poly1305@openssh.com` offers both encryption and
authentication.  As such, no separate MAC is required.  If the
`chacha20-poly1305@openssh.com` cipher is selected in key exchange, the
offered MAC algorithms MUST be ignored and no common MAC is required to
be present.

# Detailed Construction

The `chacha20-poly1305@openssh.com` cipher requires 512 bits of key
material as output from the SSH key exchange.  This forms two 256 bit
keys (`K_main` and `K_header`), used by two separate instances of
ChaCha20.

`chacha20-poly1305@openssh.com` does not require any "Initial IV" or
"Integrity" key material.

ChaCha20 requires a 8-byte nonce and a 8-byte counter.  The nonce is
constructed by encoding the packet sequence number (a 32-bit value, see
section 6.4 of [@!RFC4253]) as uint64 under the SSH wire encoding rules
given in section 5 of [@!RFC4251].  The counter is initialized with zero
unless otherwise noted and is stored in little-endian order.

The instance keyed by `K_header` is a stream cipher that is used only to
encrypt the 4 byte packet length field.  The second instance, keyed by
`K_main`, is used in conjunction with Poly1305 to build an Authenticated
Encryption with Associated Data (AEAD) that is used to encrypt and
authenticate the entire packet.

The AEAD is constructed as follows: for each packet, generate a Poly1305
key by taking the first 256 bits of ChaCha20 stream output generated
using `K_main`.  The next 256 bits of the ChaCha20 stream output are
discarded, i.e. the `K_main` ChaCha20 block counter is then set to the
little-endian encoding of 1 (i.e. {1, 0, 0, 0, 0, 0, 0, 0}) and this
instance is used for encryption of the packet payload.

The additional authenticated data (AAD) consists of the ciphertext of
the packet length and is always 4 bytes long.

The Poly1305 MAC is calculated over the concatentation of the AAD and
the ciphertext of the packet, i.e. the whole packet apart from the MAC.
The Poly1305 `mac_length` is 16 bytes.

# Packet Handling

## Padding

In openssh Encrypt Then Mac (ETM) MAC algorithms pad the packet without
the packet length field; `chacha20-poly1305@openssh.com` is an ETM
algorithm in this sense.

`chacha20-poly1305@openssh.com` is a stream cipher and has no block size
requirements; SSH therefore requires an 8-byte alignment (see section 6
of [@!RFC4253]).

The minimum size of a packet (including the `packet_length` field but
not the mac) is 12 bytes: the content is padded to at least 8 bytes plus
4 bytes for the `packet_length` field.

## Receiving packets

When receiving a packet, the length must be decrypted first.  When 4
bytes of ciphertext length have been received, they may be decrypted
using the `K_header` key.

Once the entire packet has been received, the MAC MUST be checked before
decryption.  A per-packet Poly1305 key is generated as described above
and the MAC tag calculated using Poly1305 with this key over the
ciphertext of the packet length and the payload together.  The
calculated MAC is then compared in constant time with the one appended
to the packet.

If the calculated MAC does not match the one appended to the packet
decryption must fail.  The packet MUST NOT be completely or partially
decrypted before the MAC was checked.

If the MAC was checked successfully the packet is decrypted using
ChaCha20 as described above (with `K_main` and a starting block counter
of 1).

## Sending packets

To send a packet, first encode the 4 byte length and encrypt it using
`K_header`.  Encrypt the packet payload (using `K_main`) and append it
to the encrypted length.  Finally, calculate a MAC tag and append it.

# Rekeying

ChaCha20 must never reuse a {key, nonce} for encryption nor may it be
used to encrypt more than 2^70 bytes under the same {key, nonce}.  The
SSH Transport protocol [@!RFC4253] recommends a far more conservative
rekeying every 1GB of data sent or received.  If this recommendation is
followed, then `chacha20-poly1305@openssh.com` requires no special
handling in this area.

# Differences to RFC 7539

In section 2.3 of [@RFC7539] a different ChaCha20 construction is
described; it uses a 12-byte nonce and a 4-byte counter.  By removing
the last 4 bytes of the 8-byte counter (which are always zero in ssh)
and prefixing 4 zero bytes to the 8-byte nonce one can get a [@RFC7539]
compatible representation as 4-byte counter and 12-byte nonce.

The data Poly1305 is applied to in section 2.8 of [@RFC7539] differs
too: it adds padding and encodes the length of ciphertext and AAD.

# Security considerations

Two separate cipher instances are used here so as to keep the packet
lengths confidential but not create an oracle for the packet payload
cipher by decrypting and using the packet length prior to checking the
MAC.  By using an independently-keyed cipher instance to encrypt the
length, an active attacker seeking to exploit the packet input handling
as a decryption oracle can learn nothing about the payload contents or
its MAC (assuming key derivation, ChaCha20 and Poly1305 are secure).

As the MAC needs to be checked before decrypting the packet, packet
sizes must be limited to what an implementation can buffer.

Although the Poly1305 construction is different from [@RFC7539] it
should offer comparable security.  The length of the AAD doesn't need to
be encoded as it has a fixed length of 4 bytes, and the length of the
encrypted content doesn't need to be encoded either as Poly1305 appends
a 0x01 byte to each block, leading to a fixed termination of a
potentially partial last block.

For further details on the Chacha20 and Poly1305 combination see section
4 of [@!RFC7539].

# IANA considerations

As the name `chacha20-poly1305@openssh.com` is a local extension it
cannot be registered by the IANA according to section 4.6.1 of
[@!RFC4250].

## In case this becomes "chacha20-poly1305"

Consistent with Section 4.6 of [@!RFC4250], this document registers the
name "`chacha20-poly1305`" in the Encryption Algorithm Names registry
for the encryption algorithm defined in this document.

# Acknowledgements

Markus Friedl helped on the design.

## In case this becomes "chacha20-poly1305"

This document describes the algorithm perviously known as
`chacha20-poly1305@openssh.com`.

Markus Friedl helped on the design of `chacha20-poly1305@openssh.com`.

<reference anchor='ChaCha' target='http://cr.yp.to/chacha/chacha-20080128.pdf'>
    <front>
        <title>ChaCha, a variant of Salsa20</title>
        <author initials='D.J.' surname='Bernstein' fullname='Daniel J. Bernstein'>
            <organization>The University of Illinois at Chicago</organization>
            <address>
                <email>djb@cr.yp.to</email>
                <uri>http://cr.yp.to/</uri>
            </address>
        </author>
        <date year='2008'/>
    </front>
</reference>

<reference anchor='Poly1305' target='http://cr.yp.to/mac/poly1305-20050329.pdf'>
    <front>
        <title>The Poly1305-AES message-authentication code</title>
        <author initials='D.J.' surname='Bernstein' fullname='Daniel J. Bernstein'>
            <organization>The University of Illinois at Chicago</organization>
            <address>
                <email>djb@cr.yp.to</email>
                <uri>http://cr.yp.to/</uri>
            </address>
        </author>
        <date year='2005'/>
    </front>
</reference>

{backmatter}

# Copying conditions

Regarding this entire document or any portion of it, the authors make no
guarantees and are not responsible for any damage resulting from its
use.  The authors grant irrevocable permission to anyone to use, modify,
and distribute it in any way that does not diminish the rights of anyone
else to use, modify, and distribute it, provided that redistributed
derivative works do not contain misleading author or version
information.  Derivative works need not be licensed under similar terms.

# Example

## Input

~~~
sequence Number: 0
Key Material:
000  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 ................
016  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 ................
032  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 ................
048  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 01 ................

payload:
000  15                                              .
~~~

## Intermediate results

~~~
K_main:
000  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 ................
016  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 ................

K_header:
000  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 ................
016  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 01 ................

padding ("random" values):
000  00 01 02 03 04 05                               ......

packet_content:
000  06 15 00 01 02 03 04 05                         ........

packet_length:
000  00 00 00 08                                     ....

packet:
000  00 00 00 08 06 15 00 01 02 03 04 05             ............

K_header output (first 4 bytes):
000  45 40 f0 5a                                     E@.Z

Poly1305 key (first 32 bytes of K_main output):
000  76 b8 e0 ad a0 f1 3d 90 40 5d 6a e5 53 86 bd 28 v.....=.@]j.S..(
016  bd d2 19 b8 a0 8d ed 1a a8 36 ef cc 8b 77 0d c7 .........6...w..

The 32 bytes following the Poly1305 key in the K_main output are
discarded.

Further K_main output to encrypt packet_content with using XOR:
000  9f 07 e7 be 55 51 38 7a 98 ba 97 7c             ....UQ8z...|

XOR stream to encrypt packet with:
000  45 40 f0 5a 9f 07 e7 be 55 51 38 7a 98 ba 97 7c E@.Z....UQ8z...|
~~~

## Send data

~~~
Encrypted packet:
000  45 40 f0 52 99 12 e7 bf 57 52 3c 7f             E@.R....WR<.

MAC:
000  66 02 20 17 cf ef d3 27 8a c1 3f 40 f8 52 3f af f. ....'..?@.R?.
~~~

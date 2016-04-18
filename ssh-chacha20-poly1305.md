%%%
title = "The chacha20-poly1305 authenticated encryption cipher"
abbrev = "SSH chacha20-poly1305"
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

This document describes the `chacha20-poly1305` authenticated encryption
cipher for use in SSH based on AEAD_CHACHA20_POLY1305 described in
[@!RFC7539].

{mainmatter}

# Introduction

`AEAD_CHACHA20_POLY1305` from [@!RFC7539] is an "Authenticated
Encryption with Associated Data" (AEAD) cipher.  It takes a 256-bit key
and a 96-bit nonce, and is based on the ChaCha20 and Poly1305 primitives
both designed by Daniel Bernstein.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in
[@!RFC2119].

# Negotiation

The `chacha20-poly1305` encryption algorithm offers both encryption and
authentication.  As such, no separate MAC is required.  If the
`chacha20-poly1305` algorithm is selected in key exchange, the offered
MAC algorithms MUST be ignored and no MAC is required to be present.

# Detailed Construction

The `chacha20-poly1305` cipher requires 256 bits (32 bytes) of key
material as output from the SSH key exchange.

`chacha20-poly1305` does not require any "Initial IV" or "Integrity" key
material.

The nonce is constructed by appending the big-endian encoding of the
32-bit sequence number to 8 zero bytes.

Section 2.6 in [@!RFC7539] describes how the Poly1305 key is generated
from the output of the first (counter = 0) ChaCha20 block for a given
packet.  The second half of this block ("The other 256 bits") are
discarded usually.  The next 32 bits of those otherwise discarded 256
bits are used as `packet_length_encryption`.

The `packet_length` is encrypted by performing XOR of the `packet_length`
with the `packet_length_encryption`.

The `chacha20-poly1305` encryption algorithm also serves as the MAC
algorithm, and uses the 128-bit tag from Poly1305 as MAC.

## Additional Authenticated Data

The additional authenticated data (AAD) consists of the encrypted packet
length.

# Packet Handling

## Padding

With AEAD algorithms the `packet_length` field is usually not encrypted
using the normal AEAD algorithm, as it needs to be decrypted without
knowing how long the packet is going to be.

While the `packet_length` can be encrypted in a different way, the
packet content must be padded without the `packet_length` field to a
multiple of the block size.

As `chacha20-poly1305` is a stream cipher and has no block size
requirements the minimum SSH alignment requirement of 8 bytes is used
instead (see section 6 of [@!RFC4253]).

The minimum size of a packet (including the `packet_length` field but
not the mac) is 12 bytes (instead of the 16 bytes): the content is
padded to at least 8 bytes plus 4 bytes for the `packet_length` field.

## Receiving packets

When receiving a packet the length must be decrypted first.  When 4
bytes of ciphertext length have been received, they need to be decrypted
by applying XOR with the `packet_length_encryption` key for this packet.

The receiver SHOULD verify the packet length (minimum length is 8 bytes,
must be a multiple of 8 and must not exceed maximum expected packet
length).

Once the entire packet content of further `packet_length` bytes and the
mac has been received, the packet is decrypted as described in
[@!RFC7539].

## Sending packets

To send a packet the `packet_length` is encrypted by applying XOR with
the `packet_length_encryption` key for this packet.  Then the packet
content is encrypted as described in [@!RFC7539] and the Poly1305 tag is
appended as `mac`.

# Rekeying

`AEAD_CHACHA20_POLY1305` must never reuse a {key, nonce} combination for
encryption.  The nonce wraps around every 2^32 packets, and therefore
the same key MUST NOT be used for more than 2^32 packets.  The SSH
Transport protocol [@!RFC4253] recommends a far more conservative
rekeying every 1GB of data sent or received.  If this recommendation is
followed, then `chacha20-poly1305` requires no special handling in this
area.

# Differences to RFC 7539

In section 2.3 of [@RFC7539] a different ChaCha20 construction is
described; it uses a 12-byte nonce and a 4-byte counter.  By removing
the last 4 bytes of the 8-byte counter (which are always zero in ssh)
and prefixing 4 zero bytes to the 8-byte nonce one can get a [@RFC7539]
compatible representation as 4-byte counter and 12-byte nonce.

The data Poly1305 is applied to in section 2.8 of [@RFC7539] differs
too: it adds padding and encodes the length of ciphertext and AAD.

# Security considerations

An attacker can flip single bits in the `packet_length` field by
flipping them in the encrypted data.  This is also true for other
encryption algorithms like "`*-ctr`" from [@RFC4344] based on XOR.

For further details on the Chacha20 and Poly1305 combination see section
4 of [@!RFC7539].

# IANA considerations

Consistent with Section 4.6 of [@!RFC4250], this document registers the
name "`chacha20-poly1305`" in the Encryption Algorithm Names registry
for the encryption algorithm defined in this document.

# Acknowledgements

The designed is based on `chacha20-poly1305@openssh.com` and
[@!RFC7539].

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

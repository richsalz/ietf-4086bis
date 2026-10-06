---
title: "On Random Numbers"
abbrev: "4086bis"
docname: draft-rsalz-4086bis-latest
submissiontype: IETF
category: bcp
ipr: trust200902
area: Security
workgroup: SAAG Working Group
stand_alone: yes
smart_quotes: no
obsoletes: 4086
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: D. Miller
    name: Damien Miller
    organization: OpenSSH
    email: djm@openssh.org
 -
    ins: R. Salz
    name: Rich Salz
    organization: Akamai Technologies, Inc.
    email: rsalz@akamai.com

normative:

informative:
    OSSLCONFIG:
      title: "Notes on random number generation"
      target: https://github.com/openssl/openssl/blob/master/INSTALL.md#notes-on-random-number-generation
    RANDBYTES:
      title: "RAND_bytes"
      target: https://docs.openssl.org/master/man3/RAND_bytes/
    BCRYPT:
     title: "BCryptGenRandom function (bcrypt.h)"
     target: https://learn.microsoft.com/en-us/windows/win32/api/bcrypt/nf-bcrypt-bcryptgenrandom
    ARC4RAND:
      title: "arc4random manual page"
      target: https://man7.org/linux/man-pages/man3/arc4random.3.html
    A4USRC:
      title: "arc4random_uniform source"
      target: https://github.com/openbsd/src/blob/master/lib/libc/crypt/arc4random_uniform.c
      author:
      -
        name: "Damien Miller"
    GETRAND:
      title: "getrandom manual page"
      target: https://man7.org/linux/man-pages/man2/getrandom.2.html
    TWIST:
      title: "Marsenne Twister"
      target: https://en.wikipedia.org/wiki/Mersenne_Twister
    NISTDRBG:
      title: "Recommendation for Random Number Generation Using Deterministic Random Bit Generators"
      target: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-90Ar1.pdf
      date: "June 2015"
      author:
      -
        ins: E. Barker
        name: Elaine Barker
      -
        ins: J. Kelsey
        name: John Kelsey


--- abstract

Things have changed a great deal in the two decades since RFC 4086,
"Randomness Requirements for Security," was published.
In addition, as more IETF protocols use cryptography, the need
for good-quality randomness has greatly increased.

This document provides a definition of relevant terms and
recommendations for best practices at the time of writing.


--- middle

# Introduction

Things have changed a great deal in the two decades since RFC 4086,
"Randomness Requirements for Security," was published.
The cryptographic community has greatly advanced its knowledge of
the requirements and desirable properties for random number generators,
the algorithms that underpin them, how they may be used, and how they have
been attacked.
In addition, as more IETF protocols use cryptography, the need
for good-quality randomness has greatly increased.

This document provides a definition of relevant terms and
recommendations for best practices at the time of writing.

## Structure of this Document

This document first defines some commonly-used terms in the
generation of random numbers.
This is followed by a short section that uses those terms to
define a best practice for generating random numbers.
This is followed by a section that lists some common concerns
and mistakes, and may be thought of as a "Security Considerations"
guide for implementors.
Finally, the document concludes with the standard IETF boilerplate
sections.

# Conventions and Definitions

The following sub-sections define commonly-used terms.
These are often mis-used, so the goal is to provide a common understanding.
All of the definitions below should be taken in the context of cryptography.

## Entropy

When used in information science,
entropy is the amount of information, expressed in units of bits, that is
unknown to some other party (such as an attacker) in some scenario.  For
example, to someone having no information about the actual outcome, "a value
chosen by a single flip of an ideal coin" would present an entropy of 1.0
bits, while "a sequence of 256 such flips" would present 256.0 bits.  A
variable or protocol field of `N` bits can never represent more than `N` bits of
entropy. The effective entropy encoded in actual data is often significantly
less.

Sufficient entropy is a necessary (but not sufficient) property for a secure
random number generation system. Without sufficient entropy the system may be
predictable.

Good sources of entropy include values generated from hardware, such as
disk or network timings.
Common server systems often repeat the same actions every time the boot,
which means that system-provided entropy might not be immediately
available.
Many hardware systems provide RNG facilities that may be used as
entropy sources, such as `rdrand`/`rdseed` on X86 class CPUs, and
`RNDR` registers on
ARM CPUs.

Combination of multiple entropy sources is desirable where possible, to
avoid failure if one of them turns out to be predictable.

## Seed

A seed is the specific value used to initialize an algorithm that
generates a sequence of unpredictable "random" numbers.
The seed value should have high entropy.
It is important that the seed not be disclosed, as an adversary
could duplicate the bytes generated by the algorithm and determine
any keys generated.

## Nonce

A nonce is a number that is used once. It need not be secret, and is
often public or part of the protocol. For example, QUIC defines a nonce
that is used to detect if a packet has already been received.

The non-repeatability can be highly important. For example, if AES-GCM
repeats a nonce, an adversary can determine the key. AES-SIV is
resistant to this. It is common for a nonce to be a simple incrementing
counter.

In many protocols, nonce values are sent in cleartext. For example,
the initial SSH key exchange (RFC 4253 section 7.1) includes a 16 byte
"cookie" that each peer sends to make each key exchange (statistically)
unique. A nonce that directly uses the RNG output, as opposed to
hashing it, therefore represent one
path by which an attacker may directly
observe the raw output of a random number system.

## Random Bit Generator (RBG)

A device or algorithm that produces a sequence of bits that have
the following two characteristics:

- It is statistically independent: knowing one bit provides no
information about the value of any other bit; and
- It is unbiased: no value is more likely to occur than any other value.
See {{uniform}} for concerns about bias.

## Deterministic Random Bit Generator (DRBG)

An RBG that uses a seed and produces random bits.
The security of the stream requires that the the seed is
not known by an adversary.
The output stream has a defined limit, and the DRBG will need to be
provided new seed material when the limit is reached.
See {{NISTDRBG}} for more complete specification and algorithm
descriptions.

## Pseudo-Random Number Generator (PRNG)

An older term for DRBG, although it can imply that the seed need
not be kept private, such as when using the output stream for simulations.

##  Backtracking resistance

If an adversary knows the state of the RBG at a time `T`, they will be unable
to recover the state at time `T-1`.  Further, all output up to time `T-1`
cannot be distinguished from random output.  This is usually accomplished by
ensuring that the RBG generation algorithm is a one-way function.

Put another way, backtracking resistance means that a compromise of the RBG
internal state has no effect on the security of prior outputs.
This is commonly called "forward secrecy" in protocols such as TLS.

## Forward or Prediction resistance

If an adversary knows the state of the RBG at a time `T`, they will be unable
to predict the output at a time, `T+1`.  This can only be provided only by
ensuring that a RBG is reseeded between consecutive requests, provided that
knowledge of the current RBG internal state does not allow an adversary any
useful knowledge about future RBG internal states or outputs.

# Recommendations

Use an appropriate function from the local operating system if available.
At the time of writing,
on Windows use the `BCryptGenRandom()` function described in {{BCRYPT}}.

For OpenBSD 2.1, FreeBSD 3.0, NetBSD 1.6, DragonFly 1.0, or Linux C
library since July 2022 use the `arc4random()` described in {{ARC4RAND}}.
The `getrandom()` function is also available on many systems and
is described in {{GETRAND}}.

On older Unix-like systems, the `/dev/random` or `/dev/urandom`
pseudo-devices may be available; check the documentation.
Historically, he primary difference is that the first would block if the kernel
believed there is not enough entropy in the seed material; this may no
longer be true.

If the operating system does not provide something suitable, use an
OpenSSL function
from the `RAND_bytes` set described in {{RANDBYTES}}, particularly if provided
as part of the operating system distribution as it is most likely to
enable the best source of entropy for seeding.
If the library must be configured and compiled directly,
see the notes in {{OSSLCONFIG}} about
random number generation.

If feasible, use a main DRBG to seed two separate DRBG's: one to generate
private keys, and one for all other uses.

For smaller systems that are not generating cryptographic material,
the Mersenne Twister PRNG may be acceptable.
A full description and sample code can be found at {{TWIST}}.

# Concerns {#concerns}

This section details some likely concerns and issues to consider.

## Reseeding

A DRBG needs to be reseeded with additional entropy. The same sources
used to provide the intial entropy can often be used in reseeding.
The reseeding requirements depend on the DRBG implementation details;
{{NISTDRBG}} provides an overview and some specifics. This is generally
not necessary if the random bits are provided directly by the operating system.

## Boot-time

When a system boots, or re-boots, the hardware used (or measured) to provide
the seed material is often in the same state every time. This leads to repeated
bitstreams across reboots. It is tempting to store seed material in local
storage and use it at system start-up. If that file is accessible to
an adversary, the stream of bits can be predictable.

## Fork

It's common for a server to fork a separate client process for each
incoming connection, or pre-create a pool to handle client requests.
Reset the RNG when forking.

## Uniform distribution {#uniform}

Modulo bias is a statistical distortion that happens when mapping a
a larger range of random numbers into a smaller range using a
modulo operation such as C's `%` operator.
For example, mapping the eight values `[0 .. 7]` to
the five values `[0 .. 4]` will be distorted because four is the
only value produced from only one input. This is a concern when the
upper bound isn't a power of two.

Use the `arc4random_uniform()` function if it is available.
Freely-available source can be found at {{A4USRC}}.

## Hardware

Adam Shostack: when is the hardware RNG, washed through a hash function with
some other stuff, not sufficient?

# Security Considerations

This is an important document!

# IANA Considerations

This document has no IANA actions.

# Change Log

- Draft 1:
Various clarifying edits by Dan Wing.
Remove suggestions to copy text from RFC 4086.
Rewrite entropy definition (Marsh Ray).
Don't mention mouse as an entropy source; placeholder for Hardware RNG
considerations (Adam Shostack).
Give historic difference between /dev/*random (Stephen Farrell)

- Draft 0: Published, asked for DISPATCH and CC'd SAAG.

--- back

# Acknowledgments

TODO

    ZIP: Unassigned
    Title: FROST Member Backup Strings for Shielded Zcash
    Owners: Khadim <openpgp-wallet@protonmail.com>
            aryaethn <aryaethn@gmail.com>
    Credits: Hanh (hhanh00), for the zkfrost string in Zkool
    Status: Draft
    Category: Wallet
    Created: 2026-10-26
    License: MIT
    Discussions-To: <https://github.com/zcash/zips/issues/1413>
    Pull-Request: <https://github.com/zcash/zips/pull/1432>


# Terminology

The key words "MUST", "MUST NOT", "SHOULD", and "SHOULD NOT" in this document
are to be interpreted as described in BCP 14 [^BCP14] when, and only when,
they appear in all capitals.

The terms "Mainnet" and "Testnet" in this document are to be interpreted as
defined in the Zcash protocol specification [^protocol-networks].

The terms below are to be interpreted as follows:

Group

: A set of $n$ participants holding a FROST threshold key for Orchard-protocol
  spend authorization [^zip-0312], any $t$ of whom can sign.

Member

: One participant of a group, identified by its FROST identifier.

Signing share

: The member's secret share of the group's Spend authorizing key, as output
  by key generation.

Group verifying key, $\mathsf{ak}$

: The FROST group public key, used as the Spend validating key
  $\mathsf{ak}$ of the group's Orchard full viewing key. Its encoding has a
  $\tilde{y}$ bit of $0$, as § 4.2.3 of the protocol specification requires
  [^protocol-updated-orchardkeycomponents].

$\mathsf{sk}$

: The 32-byte value the members agree on during key generation, from which
  $\mathsf{nk}$, $\mathsf{rivk}$ and, under ZIP 2005, $\mathsf{qsk}$ and
  $\mathsf{qk}$ are derived [^zip-2005-usagewithfrost].

Derivation

: How $\mathsf{nk}$ and $\mathsf{rivk}$ are obtained from $\mathsf{sk}$ and
  $\mathsf{ak}$. This ZIP knows two, defined in [Derivations].

Verifying share

: The public key of one member's signing share: the signing share multiplied
  by the Orchard spend authorization base point
  $\mathcal{G}^{\mathsf{Orchard}}$ [^protocol-concretespendauthsig].

Payload

: The bytes that hold one member's backup, in one of the versions defined in
  this ZIP.

Member backup string

: The text form of a payload, defined in [Encoding].

Envelope

: An optional binary encoding of a payload, which can also carry items after
  a version 2 payload, defined in [Envelope]. An envelope is not encrypted.

Item

: Optional data that an envelope carries after a version 2 payload, defined in
  [Items].

Age file

: A file in the format `age-encryption.org/v1` [^age]. The terms file key,
  recipient, identity and stanza below are those of that format.

File key

: The 16 random bytes that an age file is encrypted under. Each age file has
  its own.

Recipient

: A public key for which an age file is written.

Identity

: The secret key of a recipient. It opens the age files written for that
  recipient.

Stanza

: An entry of the header of an age file that holds the file key encrypted for
  one recipient. Its type says how it is encrypted. An age file written for
  several recipients has one stanza for each, and the identity of any one of
  them opens it.

Backup file

: An age file whose plaintext is an envelope, defined in [Backup file].

Wallet seed

: The seed $S$ of a ZIP 32 wallet [^zip-0032-wallet-seeds], from which the
  wallet derives its spending keys.

Backup mnemonic

: 128 random bits made only to open backup files, which a user keeps as 13
  words, defined in [Backup mnemonic].

Encryption key

: The identity that opens a backup file. It is derived from a wallet seed or
  from a backup mnemonic, as [Encryption key] specifies.

Encoder

: Software that writes a payload, a member backup string, an envelope or a
  backup file.

Decoder

: Software that reads a payload, a member backup string, an envelope or a
  backup file, and gives the fields of the payload and the items of an
  envelope, as [Decoding] specifies.

Wallet

: Software that holds a member. It uses an encoder to export the member and a
  decoder to restore it.


# Abstract

This ZIP defines a backup string for one member of a FROST group that spends
shielded ZEC with Orchard-protocol keys (in the Orchard or Ironwood pool). The
string holds the member's identifier, the threshold, the group size, the
member's signing share and the group's $\mathsf{sk}$. Together with the group's
unified full viewing key, it is enough to rebuild the member in any wallet that
implements this ZIP, without the other members being online. Restoring checks
the string against the viewing key, which both detects a string from another
group and tells the wallet which key derivation the group uses.

Version 1 of the payload is byte-for-byte the `zkfrost` string that Zkool
already ships, so existing backups stay valid.

Version 2 of the payload also holds the group's viewing key or the means to
derive it, the network and a birthday height, so that it restores on its own.

How a wallet stores or moves a payload is the wallet's choice. This ZIP
defines three optional encodings of a payload for the wallets that want them:
the string, which holds a payload and nothing else; an envelope, which holds a
payload in the clear; and a backup file, an age file that holds an envelope
encrypted for post-quantum hybrid recipients (ML-KEM-768 with X25519). After a
version 2 payload, an envelope can carry items: optional data such as the
public commitment from which every member's verifying share is computed.

The key that opens a backup file is derived from the user's wallet seed, so
that a user who keeps a seed has nothing new to keep, or from a backup
mnemonic of 13 words made for backups only.

The string is more sensitive than the signing share it carries: it also
carries $\mathsf{sk}$, from which a holder can compute the group's quantum
spending key $\mathsf{qsk}$ [^zip-2005]. It never appears on chain.


# Motivation

Several wallets now offer FROST groups for shielded ZEC: Zkool, Zafe,
Konclave, and the Zcash Foundation's tools. Each one backs up its members in its
own format, so a member can only restore their seat in the app that created
it. In July 2026, Zkool's author described the situation: "There is no
standard for it. I choose a way that makes it restorable on Zkool only. One
day, someone will write a ZIP." [^zkool-forum]

Since then, Zkool has moved to a minimal backup string (`zkfrost`, Zkool
6.35.0) that holds only what a member needs. This ZIP takes that string as its
first version, so that:

- a user can move a seat from one wallet to another, or restore it in a
  different app after losing a device;
- a group can mix implementations, which avoids having every signer come from
  one vendor;
- funds stay reachable if one app is abandoned.

ZIP 2005 adds a reason. To recover notes with the Recovery Protocol, "it is
necessary to retain, or be able to rederive, $\mathsf{sk}$, and also to know
whether $\mathsf{use\_qsk}$ was used to derive the $\mathsf{ivk}$ and
addresses" [^protocol-updated-orchardkeycomponents]. A backup format is where
that knowledge has to live.


# Privacy Implications

The string never leaves the member's custody in normal use and is not
published on chain, so this ZIP changes nothing for on-chain privacy.

Combined with the group's unified full viewing key, a string gives no viewing
capability beyond what that viewing key already gives. Without the viewing
key, $\mathsf{sk}$ yields $\mathsf{nk}$ but not the incoming viewing key, which
also depends on $\mathsf{ak}$. The string does reveal $t$, $n$ and the member's
identifier.

A version 2 payload gives the group's full viewing key on its own: it holds
$\mathsf{ak}$ with $\mathsf{sk}$, or the viewing key itself. It also names the
network and a birthday height. The items of an envelope can say more about the
group: the group commitment gives every member's verifying share, and a
registered item holds what its registrant's specification defines. All of
this is as private as the payload that they come with.

A backup file hides its envelope. A stanza does not identify its recipient,
so the file does not show who can open it. It shows the number of its
stanzas, and the length of the envelope it holds; with a group commitment
item, that length grows with the threshold $t$.

The name of a backup file can show 8 characters of the seed fingerprint of
the wallet seed that opens it (see [File name of a backup file]). These
characters link the backup files of one wallet seed, and anyone who knows
that seed's fingerprint recognises its files. A backup file named for a
backup mnemonic shows only that a mnemonic opens it.

A remaining part kept in the clear shows the fields that
[Keeping the secret fields apart] lists. An encrypted
remaining part shows nothing but its length, since its salt is random.

A wallet that shows the string as a QR code exposes it to cameras and
onlookers, as with any secret displayed on screen.


# Requirements

- A member can be rebuilt from its string and the group's unified full viewing
  key, with no other member online.
- A wallet detects a string that belongs to another group before using it.
- The restore tells which derivation the group uses.
- The format does not depend on how $\mathsf{sk}$ was agreed.
- Backups made with Zkool's `zkfrost` string stay valid.
- The format can gain new versions without breaking existing strings.
- A member can be rebuilt from a version 2 payload alone.
- A backup can carry public data of the group, and data of one wallet, next to
  the payload, without a new version of the payload for each addition. A
  wallet that does not know such data still restores the member.
- A payload can be stored encrypted, and the encryption can change without
  changing the payload.
- An encrypted backup stays secret against an adversary that records it now
  and can compute discrete logarithms later.
- A user can open an encrypted backup with the wallet seed they already keep,
  or with a secret made for backups only.


# Non-requirements

- Key generation, including how the members agree on $\mathsf{sk}$
  [^zip-0312-key-generation].
- Signing, and the transport between members.
- How a wallet stores, shows or moves a payload.
- Encryption under a passphrase (see [Open issues]).
- A birthday height in version 1. ZIP 316 lets a unified full viewing key
  carry one (typecode $\mathtt{0xC0}$) [^zip-0316-birthday], and a version 1
  restore already needs that key.
- The layout of the items that this ZIP registers to a wallet (see
  [Registered items]).
- How the other members recognise a restored member.
- Splitting $\mathsf{qsk}$ between members.
- FROST groups using Sapling keys.


# Specification

## Derivations

Let $\mathsf{H^{ask}}$, $\mathsf{H^{nk}}$, $\mathsf{H^{rivk}}$,
$\mathsf{H^{qsk}}$, $\mathsf{H^{qk}}$ and
$\mathsf{H}^{\mathsf{rivk\_ext}}_{\mathsf{qk}}$ be as defined in § 4.2.3
of the protocol specification as updated by ZIP 2005
[^protocol-updated-orchardkeycomponents].

ZIP 2005 derivation

: $\mathsf{nk} = \mathsf{H^{nk}}(\mathsf{sk})$, and
  $\mathsf{rivk} = \mathsf{H}^{\mathsf{rivk\_ext}}_{\mathsf{qk}}(\mathsf{ak}, \mathsf{nk})$
  where $\mathsf{qk} = \mathsf{H^{qk}}(\mathsf{H^{qsk}}(\mathsf{sk}))$. This is
  the $\mathsf{use\_qsk} = \mathsf{true}$ branch that ZIP 2005 requires for
  FROST [^zip-2005-usagewithfrost]. Notes received by such a group are
  quantum recoverable.

Legacy derivation

: $\mathsf{nk} = \mathsf{H^{nk}}(\mathsf{sk})$ and
  $\mathsf{rivk} = \mathsf{H^{rivk}}(\mathsf{sk})$, with the group verifying key
  as $\mathsf{ak}$. Zkool created groups this way before supporting ZIP 2005.
  Such a group follows neither branch of § 4.2.3
  [^protocol-updated-orchardkeycomponents]: $\mathsf{ak}$ is not derived from
  $\mathsf{sk}$, and $\mathsf{rivk}$ does not commit to a $\mathsf{qk}$. Notes
  received by such a group are therefore not quantum recoverable.

## Encoding

A member's backup is a payload: a sequence of bytes in one of the versions
defined below. The version 2 payload is the one discussed in zcash/zips#1413
[^zips-1413-v2], unchanged.

How a wallet stores, shows or moves a payload is the wallet's choice: a file,
a QR code, a container of its own. A wallet conforms whatever it puts the
payload in, as long as the bytes of the payload are unchanged. An unencrypted
payload needs nothing around it: it is the payload bytes.

This ZIP defines three encodings of a payload. All are optional. A wallet uses
one when it wants what that encoding gives:

- a member backup string, defined in this section: text with a checksum, which
  a user can copy. It holds a payload and nothing else;
- an envelope, defined in [Envelope]: bytes that hold a payload in the clear,
  and after a version 2 payload, zero or more items (see [Items]);
- a backup file, defined in [Backup file]: an age file that holds an envelope,
  encrypted for one or more recipients.

It also defines a way to keep a version 2 payload and its items in two parts,
one of which holds the two secret fields, in [Keeping the secret fields apart].
It is optional as well.

An encoder that writes a member backup string, an envelope or a backup file,
and a decoder that reads one, MUST follow the rules of that encoding.

A member backup string is the Bech32m [^bip-0350] encoding of a payload, with
the human-readable part `zkfrost`. As in ZIP 316 [^zip-0316], the length limit
of Bech32m is not applied; a version 1 string is 123 characters long, and a
version 2 string is 186 to 286 characters long.

A member backup string never carries items. Items can make a backup several
kilobytes long, which as a string would be thousands of characters, far
beyond the length that the Bech32m checksum is designed for (see
[Rationale]). An envelope carries them.

Encoders MUST produce lowercase strings. Decoders MUST accept a string that is
entirely lowercase or entirely uppercase, and MUST reject a string in mixed
case, as Bech32m requires. A wallet can still show the string in uppercase,
for instance in a QR code (see [Rationale]), by converting it after encoding.

The human-readable part is the same on every network. A version 1 payload
names no network: the network is given by the unified full viewing key used to
restore. A version 1 string paired with another group's viewing key, on either
network, fails the check in [Restore]. A version 2 payload names its network
in a field.

## Payload, version 1

| Offset | Size | Field |
|--------|------|-------|
| 0      | 1    | format version, $\mathtt{0x01}$ |
| 1      | 1    | member identifier $i$ |
| 2      | 1    | threshold $t$ |
| 3      | 1    | group size $n$ |
| 4      | 32   | signing share |
| 36     | 32   | $\mathsf{sk}$ |

The member's FROST identifier is the Pallas scalar with integer value $i$.
The group size $n$ is the number of members.

The signing share is the little-endian encoding of a Pallas scalar, as in the
FROST(Pallas, BLAKE2b-512) ciphersuite [^zip-0312]. It MUST be the share that
matches the $\mathsf{ak}$ in the group's viewing key. Key generation can output
a group key with either $\tilde{y}$ bit; when it is $1$, the group key and
every share are negated. The `reddsa` crate does this at the end of its key
generation, and frost-client does it after the DKG. An implementation that
skips this step would export a wrong share for about half of all groups.

## Payload, version 2

A version 2 payload is, with the offset of each field in its four layouts:

| Field | Size | sk, short | sk, full | no sk, short | no sk, full |
|-------|------|-----------|----------|--------------|-------------|
| format version, $\mathtt{0x02}$ | 1 | 0 | 0 | 0 | 0 |
| key source: $\mathtt{0x00}$ ZIP 2005 from $\mathsf{sk}$, $\mathtt{0x01}$ legacy from $\mathsf{sk}$, $\mathtt{0x02}$ viewing key only | 1 | 1 | 1 | 1 | 1 |
| network: $\mathtt{0x00}$ Mainnet, $\mathtt{0x01}$ Testnet | 1 | 2 | 2 | 2 | 2 |
| threshold $t$ | 1 | 3 | 3 | 3 | 3 |
| group size $n$ | 1 | 4 | 4 | 4 | 4 |
| identifier form: $\mathtt{0x00}$ short, $\mathtt{0x01}$ full | 1 | 5 | 5 | 5 | 5 |
| identifier | 1 or 32 | 6 | 6 | 6 | 6 |
| signing share | 32 | 7 | 38 | 7 | 38 |
| birthday height | 4 | 39 | 70 | 39 | 70 |
| $\mathsf{sk}$ | 32 | 43 | 74 | | |
| $\mathsf{ak}$ | 32 | 75 | 106 | | |
| $\mathsf{ak} \,\|\, \mathsf{nk} \,\|\, \mathsf{rivk}$ | 96 | | | 43 | 74 |
| **length** | | **107** | **138** | **139** | **170** |

Key sources $\mathtt{0x00}$ and $\mathtt{0x01}$ use the "sk" layouts,
$\mathtt{0x02}$ the "no sk" ones. The short identifier is the integer $i$; the
full one is the canonical little-endian Pallas scalar. The birthday is a
little-endian block height on the payload's network, 0 if unknown.

The key source says where $\mathsf{nk}$ and $\mathsf{rivk}$ come from, and
fixes the last two fields:

| Key source        | Meaning | $\mathsf{sk}$ field | Last field |
|-------------------|---------|---------------------|------------|
| $\mathtt{0x00}$   | ZIP 2005 derivation from $\mathsf{sk}$ and $\mathsf{ak}$ | 32 bytes | $\mathsf{ak}$, 32 bytes |
| $\mathtt{0x01}$   | legacy derivation from $\mathsf{sk}$ | 32 bytes | $\mathsf{ak}$, 32 bytes |
| $\mathtt{0x02}$   | viewing key only; the group has no $\mathsf{sk}$ | absent | Orchard full viewing key, 96 bytes |

The $\mathsf{ak}$ field is the 32-byte encoding of $\mathsf{ak}$ that starts a
raw Orchard full viewing key. The Orchard full viewing key field is the raw
encoding of § 5.6.4.4 of the protocol specification
[^protocol-orchardfullviewingkeyencoding]: the 32-byte encodings of
$\mathsf{ak}$, $\mathsf{nk}$ and $\mathsf{rivk}$, in this order.

The network is $\mathtt{0x00}$ for Mainnet or $\mathtt{0x01}$ for Testnet. No
value is given to Regtest: ZIPs do not specify Regtest, which is left to
implementations, and any other value fails decoding (see [Decoding]).

The threshold $t$ and the group size $n$ are as in version 1.

The identifier form is $\mathtt{0x00}$ (short) or $\mathtt{0x01}$ (full):

- short: the member identifier field is one byte, an integer $i$ from 1 to
  255. The member's FROST identifier is the Pallas scalar with integer
  value $i$.
- full: the member identifier field is 32 bytes, the little-endian encoding
  of the member's FROST identifier, a Pallas scalar.

An encoder MUST use the short form when the identifier is an integer from 1 to
255, and a decoder MUST reject a full form that would fit in the short one
(see [Decoding]), so each member has exactly one encoding. The
identifier is not compared with $n$: a group whose shares were refreshed to
remove a member keeps the other members' identifiers [^frost-refresh], so it
can have identifiers 1, 2 and 4 with $n = 3$.

The signing share is encoded as in version 1, and the same rule on its sign
applies (see [Payload, version 1]).

The birthday height is a block height on the network that the payload names,
in little-endian order. No block below it holds a transaction of the group. It
is 0 when the encoder does not know such a height.

## Envelope

An envelope is the following bytes:

| Size   | Field |
|--------|-------|
| 4      | magic: $\mathtt{0x8A}$ $\mathtt{0x5A}$ $\mathtt{0x46}$ $\mathtt{0x42}$ ($\mathtt{0x8A}$, then `ZFB` in ASCII) |
| 1      | container version, $\mathtt{0x01}$ |
| 1      | wrapper id |
| varies | wrapper parameters, fixed by the wrapper id |
| rest   | body |

The contents of an envelope are a payload, followed, when it is a version 2
payload, by zero or more items (see [Items]). The wrapper says how the body
holds them:

| Wrapper id | Wrapper | Wrapper parameters | Body |
|------------|---------|--------------------|------|
| $\mathtt{0x00}$ | none | none | the contents |
| other      | unassigned | | |

Wrapper $\mathtt{0x00}$ carries a payload and its items in the clear (see
[Rationale]). An encoder MUST NOT write an envelope under another wrapper id.
A backup file encrypts an envelope under wrapper $\mathtt{0x00}$ (see
[Backup file]).

An envelope is bytes. An encoder that writes an envelope as text MUST write
the base64url encoding (RFC 4648, section 5 [^rfc4648-base64url]) of the
envelope's bytes without padding, and nothing before or after it. A decoder
that reads text MUST ignore ASCII whitespace and line breaks anywhere in it,
MUST accept the alphabet of section 4 [^rfc4648-base64] as well as that of
section 5, with or without `=` padding, and MUST then apply the decoding of an
envelope (see [Decoding]) to the bytes. A decoder MUST reject text that is
neither a member backup string nor the base64 of an envelope.

Every envelope starts with the bytes $\mathtt{0x8A}$ $\mathtt{0x5A}$
$\mathtt{0x46}$ $\mathtt{0x42}$ $\mathtt{0x01}$, so its text starts with
`ilpGQgE` (`ilpGQgEA` under wrapper $\mathtt{0x00}$), which no member backup
string starts with.


## Items

An item is optional data that an envelope carries after a version 2 payload
(see [Envelope]). No item follows a version 1 payload, and a member backup
string carries none. An item is encoded as:

| Size   | Field |
|--------|-------|
| varies | typecode, as a $\mathtt{compactSize}$ [^Bitcoin-CompactSize] |
| varies | length of the value in bytes, as a $\mathtt{compactSize}$ |
| length | value |

A typecode is an integer from 1 to $\mathtt{0x02000000}$, the bound that ZIP
316 sets for the typecodes of its items [^zip-0316]. Every typecode that this
ZIP assigns or leaves to private use is at most $\mathtt{0xFC}$, so its
encoding is one byte.

The items take all the bytes of the envelope's body after the payload. An
encoder MUST write items in strictly increasing order of typecode, so a
typecode appears at most once, and MUST write each typecode and each length
in the shortest $\mathtt{compactSize}$ form. A body therefore has one
encoding.

A decoder MUST skip an item whose typecode it does not know. A wallet built
before an item was defined still restores the member from an envelope that
carries it.

The typecodes are divided as follows:

| Typecode | Use |
|----------|-----|
| $\mathtt{0x00}$ | never used |
| $\mathtt{0x01}$ to $\mathtt{0x7F}$ | standard items, defined by this ZIP |
| $\mathtt{0x80}$ to $\mathtt{0xBF}$ | registered items, assigned to a wallet or a protocol by the table in [Registered items] |
| $\mathtt{0xC0}$ to $\mathtt{0xFC}$ | private use, never assigned (see [Private-use items]) |
| $\mathtt{0xFD}$ to $\mathtt{0x02000000}$ | reserved, encoded in more than one byte; none is assigned |

The standard items are:

| Typecode | Item | Length of the value |
|----------|------|---------------------|
| $\mathtt{0x01}$ | group commitment | $32 \cdot (t - 1)$ |
| $\mathtt{0x02}$ to $\mathtt{0x7F}$ | unassigned | |

An encoder MUST NOT write an unassigned standard typecode, or a reserved
typecode.

### Group commitment

Key generation fixes a secret polynomial $f$ of degree $t - 1$ over the Pallas
scalar field: $f(0)$ is the group's Spend authorizing key, and
$f(i)$ is the signing share of the member whose identifier is $i$. Let
$a_0, \ldots, a_{t-1}$ be its coefficients, so that
$f(x) = \sum_{k=0}^{t-1} a_k \cdot x^k$. The group commitment is the $t$ Pallas
points

$$\Phi_k = [a_k]\, \mathcal{G}^{\mathsf{Orchard}} \quad \text{for } k = 0, \ldots, t - 1,$$

where $\mathcal{G}^{\mathsf{Orchard}}$ is the Orchard spend authorization base
point [^protocol-concretespendauthsig]. FROST key generation outputs these
points as its verifiable secret sharing commitment. A trusted dealer computes
them from the coefficients it chose [^frost-vss]. In a distributed key
generation, each participant commits to a polynomial of its own and $f$ is the
sum of these polynomials, so $\Phi_k$ is the sum of the participants' $k$-th
commitment points; COCKTAIL-DKG calls it the aggregated commitment
[^cocktail-dkg-round2].

The group commitment is taken after the sign of the group verifying key is
fixed. Key generation can output a group verifying key whose encoding has a
$\tilde{y}$ bit of $1$. In that case the group verifying key, every signing
share and every verifying share are negated (see [Payload, version 1]). This
replaces $f$ by $-f$, so every $\Phi_k$ is negated as well. The payload and
its items hold the values after this step:

- the payload's $\mathsf{ak}$ is the encoding of the group verifying key after
  the step;
- its signing share is $f(i)$ for the polynomial $f$ after the step;
- the group commitment item is the commitment of that same $f$.

An encoder MUST NOT write commitment points from before this step. The
commitment points that the participants of a distributed key generation
exchange are from before the step, so an encoder that computes the group
commitment from them MUST negate every $\Phi_k$ when the step negated the
shares.

What key generation returns does not say whether the step negated the shares:
the step has already been applied to its output. The `reddsa` crate's
`dkg::part3`, for instance, returns a group verifying key on which
`has_even_y()` is always true, whether or not the round 1 commitments need
negating. An encoder that computes the group commitment from the round 1
commitments finds out by comparing the sum of their constant terms, $\Phi_0$
from before the step, with the point that $\mathsf{ak}$ encodes: when the two
are equal, the step negated nothing; when the sum is the negation of that
point, the step negated the shares.

The group commitment in an envelope is that of the polynomial of which the
signing share of the envelope's payload is a value.

The value of the item is $\Phi_1, \ldots, \Phi_{t-1}$ in this order, each as
the 32-byte encoding $\mathsf{repr}_{\mathbb{P}}$ of a Pallas point
[^protocol-pallasandvesta]. $\Phi_0$ is not in the item, because the payload
already holds it: $\Phi_0 = [f(0)]\, \mathcal{G}^{\mathsf{Orchard}}$ is the
group verifying key, and the 32 bytes of $\mathsf{ak}$ are its encoding under
$\mathsf{repr}_{\mathbb{P}}$, with a $\tilde{y}$ bit of $0$
[^protocol-updated-orchardkeycomponents]. A decoder obtains $\Phi_0$ by
decoding these 32 bytes as a Pallas point. An encoder MUST NOT write the item
unless the $\Phi_0$ of the commitment it holds has exactly this encoding.

The verifying share of the member whose identifier is $i$ is

$$Y_i = \sum_{k=0}^{t-1} [i^k]\, \Phi_k.$$

With this item, a restored member computes the verifying share of every
member whose identifier it knows, without contacting anyone.

Since $\Phi_0$ is read from $\mathsf{ak}$ and not stored, an item that holds
the points from before the sign step, with a share from after it, cannot be
recognised by its $\Phi_0$. It is recognised by the member's own verifying
share: the restore computes $Y_i$ from $\mathsf{ak}$ and the item, and compares
it with the signing share multiplied by $\mathcal{G}^{\mathsf{Orchard}}$ (see
[Restore, version 2], step 5). With $\Phi_1, \ldots, \Phi_{t-1}$ of the wrong
sign, the two are equal only if $\sum_{k=1}^{t-1} [i^k]\, \Phi_k$ is the zero
point, which has negligible probability for a polynomial chosen at random.

The group commitment holds no secret.

### Registered items

A registered item is assigned by this table to a wallet or a protocol, the
registrant. The registrant defines the layout of the value in its own
specification. Other wallets skip the item.

| Typecode | Registrant | Item |
|----------|------------|------|
| $\mathtt{0x80}$ to $\mathtt{0x8F}$ | Zodl | reserved for the registrant; layouts in the registrant's own specification |
| $\mathtt{0x90}$ to $\mathtt{0xBF}$ | | unassigned |

An encoder MUST NOT write an unassigned registered typecode, or a typecode
assigned to another registrant unless it follows that registrant's
specification.

The registrant's specification states how sensitive the value of each of its
items is.

### Private-use items

A private-use typecode is never assigned, so two wallets can use the same one
for different data. An encoder MUST start the value of a private-use item
with a 4-byte tag chosen by the wallet that writes it. A decoder MUST skip a
private-use item whose tag is not one of its own.

## Backup file

A backup file is an age file, as the age specification defines it [^age]. Its
plaintext is an envelope under wrapper $\mathtt{0x00}$: the payload followed
by its items (see [Envelope]).

The recipients of a backup file are of the age type
`mlkem768x25519` [^age-mlkem768x25519]. Such a stanza encrypts the
file key with HPKE [^rfc9180] under the key encapsulation mechanism
MLKEM768-X25519 [^hpke-pq], also named X-Wing, which combines ML-KEM-768
[^fips203] with X25519 [^rfc7748]. The `age` tool reads and writes this type
from version 1.3. An identity of this type is 32 bytes, and its recipient
is derived from it as the age specification says.

An encoder that writes a backup file:

- MUST write an age file as the age specification says, with a file key of 16
  bytes from a cryptographic random number generator;
- MUST write as its plaintext an envelope under wrapper $\mathtt{0x00}$;
- MUST write one stanza of type `mlkem768x25519` for each recipient, and at
  least one;
- MUST NOT write a stanza of another type that gives the file key;
- MAY write stanzas of other types that give no file key. Some
  implementations of age add one such stanza, with a random type and random
  content, to every file they write, so that readers keep skipping the types
  they do not know.

The identity of any one recipient opens the file.

A decoder that reads a backup file MUST skip a stanza whose type it does not
know, as the age specification requires. It MUST reject the file if any of
the following holds:

- the file is not an age file under the rules of the age specification;
- a stanza of type `mlkem768x25519` is not well formed: it does not have
  exactly one argument that is the canonical unpadded base64 of 1120 bytes,
  or its body is not 32 bytes;
- no stanza of type `mlkem768x25519` gives the file key under the identity
  that the decoder was given;
- the header's MAC does not verify, or the payload of the age file does not
  decrypt, under that file key;
- the plaintext is not an envelope, or it is an envelope under another
  wrapper than $\mathtt{0x00}$.

The decoder then reads the plaintext as an envelope (see [Decoding]).

### Text form of a backup file

The text form of a backup file is the ASCII armor that the age
specification defines [^age-armor]. It is three parts:

1. the line `-----BEGIN AGE ENCRYPTED FILE-----`;
2. the bytes of the age file in base64, with the alphabet of section 4 of
   RFC 4648 and with `=` padding [^rfc4648-base64], in lines of 64
   characters, the last of which can be shorter;
3. the line `-----END AGE ENCRYPTED FILE-----`.

An encoder MUST end each line with a line feed, and MUST NOT write anything
before the first line or after the last.

A decoder MUST ignore ASCII whitespace before the first line and after the
last, and MUST accept a line feed or a carriage return and a line feed at the
end of each line. It MUST reject any other text: a line before the first line
or after the last, a line of the body longer than 64 characters, a line other
than the last one shorter than 64 characters, an empty line, or base64 that is
not canonical. `age` 1.3.2 refuses a file with any line before the first line
or after the last, so a backup file carries no metadata of its own.

## Encryption key

The encryption key of a backup file is an identity of type
`mlkem768x25519`. Its 32 bytes are derived from one of two secrets: a wallet
seed (see [Encryption key from a wallet seed]) or a backup mnemonic (see
[Encryption key from a backup mnemonic]). An encoder writes a backup file for
the recipient of the encryption key.

### Encryption key from a wallet seed

Let $S$ be a wallet seed of 32 to 252 bytes [^zip-0032-wallet-seeds], and
$\mathsf{coinType}$ the ZIP 32 coin type of the network that the wallet uses:
133 for Mainnet, 1 for Testnet.

The key is derived in the Account Metadata Key tree of ZIP 325
[^zip-0325-metadata-key-tree], whose $\mathsf{ContextString}$ is
$\texttt{“MetadataKeys”}$ and whose $\mathsf{ZipNumber}$ is 325, with
ZIP 32's registered key derivation [^zip-0032-registered-key-derivation], in
three steps:

1. $\mathsf{AccountMetadataKey}$ is the key of ZIP 325 for $S$,
   $\mathsf{coinType}$ and account 0:
   $m_{\mathsf{metadata}} / 325' / \mathsf{coinType}' / 0'$.
2. $K_{\mathsf{inherent}} = \mathsf{CKDreg}(\mathsf{AccountMetadataKey}, 0, [\,])$
   is its Account-level Inherent Metadata Key.
3. $(\mathsf{sk}_{\mathsf{backup}}, \mathsf{c}_{\mathsf{backup}}) = \mathsf{CKDreg}(K_{\mathsf{inherent}}, \mathtt{0x7FFFFFFF}, \texttt{“ZodlFROSTMemberBackupV1”})$
   is its Private-use Inherent Metadata Key [^zip-0325-private-use] with
   $\mathsf{PrivateUseSubject} = \texttt{“ZodlFROSTMemberBackupV1”}$.

In path notation:

    m_metadata / 325' / coin_type' / 0' / 0' / (0x7FFFFFFF', "ZodlFROSTMemberBackupV1")

The 32 bytes of $\mathsf{sk}_{\mathsf{backup}}$ are the identity. The account
is 0 whatever account the wallet uses. The coin type is in the path, so a
wallet seed gives another encryption key on each network.

A wallet MUST reject a wallet seed shorter than 32 bytes or longer than 252
bytes.

### Backup mnemonic

A backup mnemonic is 16 bytes. An encoder MUST draw them from a
cryptographic random number generator. It MUST NOT derive them from a
passphrase, or let the user choose them.

A wallet shows a backup mnemonic to its user as 13 words, in lowercase,
separated by single spaces: the word `zbackup`, then the 12 words that BIP 39
gives for the 16 bytes as entropy, with the English word list [^bip-0039].

A wallet that reads a backup mnemonic MUST ignore case, MUST treat any run of
whitespace as one separator, and MUST ignore whitespace before the first word
and after the last. It MUST accept the 12 words without the word `zbackup`
before them. It MUST reject the text if any of the following holds:

- it is not 12 words, once a first word `zbackup` is removed;
- a word is not in the BIP 39 English word list;
- the BIP 39 checksum of the 12 words does not match.

The word `zbackup` is not in the BIP 39 English word list, so a wallet given
the 13 words as a seed phrase rejects them. A wallet MUST NOT use the 12 words
as a wallet seed, and MUST NOT apply the seed derivation of BIP 39 (PBKDF2)
to them.

### Encryption key from a backup mnemonic

Let $\mathsf{m}$ be the 16 bytes of a backup mnemonic. The identity is

$$\mathsf{BLAKE2b\text{-}256}(\texttt{“ZcashFROSTAgeKey”},\, \mathsf{m}),$$

that is BLAKE2b [^protocol-concreteblake2] with a 32-byte output, the 16-byte
personalization `ZcashFROSTAgeKey`, no key and no salt parameter, applied to
the 16 bytes of $\mathsf{m}$.

### File name of a backup file

A wallet that gives a backup file a name SHOULD use one of these forms:

- `<prefix>-seed-<tag>.zbackup` when the encryption key comes from a wallet
  seed. `<tag>` is the first 8 characters, after `zip32seedfp1`, of the text
  form of the ZIP 32 seed fingerprint of that wallet seed: Bech32m with the
  human-readable part `zip32seedfp` [^zip-0032-seed-fingerprints]. These 8
  characters hold the first 40 bits of the fingerprint.
- `<prefix>-key.zbackup` when the encryption key comes from a backup
  mnemonic.

`<prefix>` is chosen by the wallet. A decoder reads a name without regard to
case. A name that ends in neither form gives no hint.

The name is a hint only. A user can rename a file, and a backup file pasted
as text has no name. A decoder MUST NOT reject a backup file for its name. It
uses the name only to choose which encryption key to try first. When the name
gives no hint, or the key it names does not open the file, the decoder tries
the other encryption keys it has.

## Decoding

The input of a decoder is a payload, a member backup string, an envelope or
a backup file. A decoder of a backup file first takes the envelope out of it
(see [Backup file]). A decoder of a string or of an envelope takes the payload
out of it, as below. Every payload is then parsed under the same rules, wherever it came
from. A decoder of an envelope also parses the items that follow a version 2
payload.

The decoder reads the input as it is given. It MUST NOT remove or skip any
byte of it, such as a space or a line break before or after a string. The
text of an envelope and the text form of a backup file are the exceptions:
their decoders ignore whitespace (see [Envelope] and
[Text form of a backup file]). A wallet
that takes a string from its user removes such characters before it calls the
decoder.

A decoder that reads a member backup string MUST reject the string if any of
the following holds:

- it is not valid Bech32m, including a bad checksum, a Bech32 checksum,
  mixed case, or a character that Bech32m does not allow, such as a space
  or a line break after the string;
- the bits left over when converting its data part to bytes are more than
  4 or not all zero, as ZIP 173 requires [^zip-0173]. A version 1 string
  leaves 1 bit, so each payload has only one lowercase string;
- its human-readable part, compared case-insensitively, is not `zkfrost`.

The payload is the bytes that the data part converts to.

A decoder that reads an envelope MUST reject it if any of the following
holds:

- it does not start with the four magic bytes of [Envelope];
- it is shorter than 6 bytes, so that it has no container version or no
  wrapper id;
- the container version is not $\mathtt{0x01}$;
- the wrapper id is not $\mathtt{0x00}$.

The contents are the body. The payload is the start of the contents: for
format version $\mathtt{0x01}$, 68 bytes; for format version $\mathtt{0x02}$,
the length that its key source and identifier form give in
[Payload, version 2]. When the contents are shorter than that, or end before
the bytes that give that length, the payload is the whole contents, and it
fails the length rule of its version below; when those bytes hold an unknown
value, the payload fails rule 4 below. The bytes after a version 1 payload
make the decoder reject the envelope. The bytes after a version 2 payload are
its items: once the payload has been decoded, the decoder MUST reject the
envelope if any of the following holds:

- an item has typecode $\mathtt{0x00}$, or a typecode greater than
  $\mathtt{0x02000000}$;
- an item's typecode is not in the shortest $\mathtt{compactSize}$ form, or
  the contents end before the end of the typecode;
- an item's typecode is not greater than the typecode of the item before it;
- an item's length is not in the shortest $\mathtt{compactSize}$ form, or the
  contents end before the end of the item's length or value;
- a group commitment item's value is not $32 \cdot (t - 1)$ bytes long, or
  holds 32 bytes that are not the encoding of a Pallas point, or holds the
  encoding of the zero point;
- a private-use item's value is shorter than 4 bytes.

A decoder that takes more than one of these forms from one input tells them
apart by the first bytes. An envelope starts with the four magic bytes, and
the text of an envelope with `ilpGQgE` (see [Envelope]). A member backup
string starts with `zkfrost1` or `ZKFROST1` in ASCII. A payload starts with
its format version, $\mathtt{0x01}$ or $\mathtt{0x02}$. A remaining part,
which is not a form of a payload and is read only together with a secret part,
starts with its own four magic bytes, and its text with `ilpGUgE` (see
[Format of the remaining part]).
A backup file starts with `age-encryption.org/v1` in ASCII, and its text
form, after any whitespace, with `-----BEGIN AGE ENCRYPTED FILE-----` (see
[Backup file]).

A decoder parses a payload in two steps:

1. Read the format version, the first byte of the payload. The decoder MUST
   reject the payload if it is empty, which fails the length rule of either
   version, or if its format version is neither $\mathtt{0x01}$ nor
   $\mathtt{0x02}$.
2. Parse the payload under the rules of its version, below.

Checking the container version before the wrapper, and the format version
before the length, lets a decoder report an envelope or a payload of a later
version as unsupported rather than as corrupt.

A decoder MUST reject a version 1 payload if any of the following holds:

- the payload is not exactly 68 bytes long;
- $n = 0$, $i = 0$, $i > n$, $t < 2$, or $t > n$. FROST needs at least two
  signers.

A decoder MUST reject a version 2 payload, or the member backup string that
holds it, if any of the following holds, checked in this order:

1. it is not valid Bech32m (including mixed case), or the leftover bits are
   more than 4 or not all zero;
2. its human-readable part, compared without regard to case, is not
   `zkfrost`;
3. the format version is not $\mathtt{0x02}$ (a version 1 payload is decoded
   under the version 1 rules above);
4. the key source, network or identifier form is not one of the values in
   [Payload, version 2];
5. the payload is not exactly the length its key source and identifier form
   give;
6. $t < 2$ or $t > n$;
7. the identifier is 0, or, in the full form, it is not a canonical Pallas
   scalar or it is an integer from 1 to 255, which the short form holds;

Rules 3 and 4 read only the bytes the payload has: a payload that ends before
the version, key source, network or identifier form fails rule 5. A truncated
payload is then reported as damaged, while an unknown value that is present
still fails rule 4.

Rules 1 and 2 are those of a member backup string, above. A version 2 payload
given as bytes, or taken from an envelope, is checked from rule 3 on.

These seven rules are those of the version 2 text discussed in
zcash/zips#1413 [^zips-1413-v2], with only their references adapted to this
document. A version 2 payload is the same, and decodes the same, whether it
is given as bytes, in a string or in an envelope.

Decoding checks the encoding and nothing else. For both versions, it does not
check that the signing share is a canonical Pallas scalar other than zero,
that $\mathsf{sk}$ is a valid Orchard spending key, that the $\mathsf{ak}$
field encodes a valid Spend validating key, that the Orchard full viewing key
field is a valid key, or that the signing share matches the group
commitment item. These are checked when the member is restored (see [Restore] and
[Restore, version 2]), so a payload that decodes is not yet one that
restores.

## Restore

This section restores a version 1 payload. [Restore, version 2] restores a
version 2 payload.

The inputs are a member backup string and the group's unified full viewing
key. The viewing key MUST contain an Orchard component (typecode
$\mathtt{0x03}$ [^zip-0316]), which holds $\mathsf{ak}$, $\mathsf{nk}$ and
$\mathsf{rivk}$. Ironwood uses the same Orchard-protocol keys.

1. Decode the string as above.
2. Reject the string if $\mathsf{sk}$ is not a valid Orchard spending key,
   that is if $\mathsf{H^{ask}}(\mathsf{sk}) = 0$, or if the signing share is
   not the canonical encoding of a Pallas scalar. (A group's $\mathsf{ask}$ is
   not derived from $\mathsf{sk}$; the first check only keeps decoders in
   agreement with the validity rule for Orchard spending keys.)
3. Take $\mathsf{ak}$ from the viewing key. Compute $\mathsf{nk}$ and
   $\mathsf{rivk}$ under each derivation of [Derivations].
4. If the ZIP 2005 derivation gives the viewing key's $\mathsf{nk}$ and
   $\mathsf{rivk}$, the group follows the ZIP 2005 derivation. Otherwise, if
   the legacy derivation does, the group follows the legacy derivation.
   Otherwise the wallet MUST reject the string: it belongs to another group.
5. Rebuild the member's key material from the identifier, the signing share,
   the threshold $t$ and $\mathsf{ak}$ as the group verifying key. If the
   wallet knows the group's verifying shares, it SHOULD check that the
   member's verifying share matches.

The steps above are those of version 1 as first published. A wallet that
implements version 2 MUST also reject the string, at step 2, if its signing
share is zero. Zero is the canonical encoding of a Pallas scalar, so step 2
as written does not reject it. [Restore, version 2] rejects it at step 3.

A wallet SHOULD tell the user which derivation was found, and SHOULD warn when
it is the legacy one, whose notes are not quantum recoverable.

The check in step 4 proves that the viewing key was derived from the string's
$\mathsf{sk}$, not that its $\mathsf{ak}$ is the group's (see
[Security Considerations]). Before it shows an address to receive funds, a
wallet SHOULD check the viewing key against a source the member trusts, such as
the export made with its own string (see [Export]), an address of the group
that the member recorded before and that the viewing key gives, or the same
viewing key obtained from at least two other members. A member that lost its
device may have lost its export too, and then has to ask the others for the
key, which is the case the attack needs: if one of the members it asks is
honest, a forged key shows up as a mismatch.

After the restore, the wallet MUST retain $\mathsf{sk}$ when the group follows
the ZIP 2005 derivation. ZIP 2005 makes $\mathsf{qsk}$, which is derived from
$\mathsf{sk}$, essential for recovering the group's notes once the current
Orchard protocol is disabled [^zip-2005-usagewithfrost], and the signing
share and viewing key alone do not give it. A wallet SHOULD also retain
$\mathsf{sk}$ for a legacy group, since without it the member cannot export
its string again.

A restored member holds no signing nonces. It MUST generate fresh ones for its
next signing session.

The check in step 4 proves $\mathsf{sk}$, not the signing share. A wrong share,
such as one stored with the wrong sign, passes the restore and is caught at the
member's first signing: the verifying shares of the $t$ signers, which travel
with their commitments, interpolate to $\mathsf{ak}$ only if every one of them
is right. The signing transport is out of scope for this ZIP, but a signing
protocol used with these strings SHOULD carry each signer's verifying share,
so that a member restored from a string is checked when it first signs.

## Restore, version 2

The input is a version 2 payload, in any of its forms, and the items of the
envelope that carried it, if any. No viewing key is needed. The steps are those of the version 2 text discussed in
zcash/zips#1413 [^zips-1413-v2-restore], with the additions that this ZIP makes
for the signing share of zero and for the group commitment item.

The wallet carries out the steps below in order. It MUST reject the payload,
and restore nothing from it, when a step says to reject.

1. Decode the payload as above.
2. Reject it if its network is not the wallet's.
3. Reject it if the signing share is not a canonical Pallas scalar or is zero,
   or, for key sources $\mathtt{0x00}$ and $\mathtt{0x01}$, if $\mathsf{sk}$ is
   not a valid Orchard spending key, that is if
   $\mathsf{H^{ask}}(\mathsf{sk}) = 0$, as in [Restore].
4. Build the full viewing key: for $\mathtt{0x02}$, take it from the payload;
   for $\mathtt{0x00}$ and $\mathtt{0x01}$, compute $\mathsf{nk}$ and
   $\mathsf{rivk}$ from $\mathsf{sk}$ and $\mathsf{ak}$ under the recorded
   derivation (see [Derivations]); no other derivation is tried. Reject the
   payload if that key is invalid under the rule for Orchard raw full viewing
   keys [^protocol-orchardfullviewingkeyencoding], that is if $\mathsf{ak}$,
   $\mathsf{nk}$ or $\mathsf{rivk}$ is not canonically encoded, if
   $\mathsf{ak}$ is not a valid affine x-coordinate of a Pallas point in
   $\mathbb{P}^*$, or if the incoming viewing key $\mathsf{ivk}$ or the
   internal incoming viewing key $\mathsf{ivk}_{\mathsf{internal}}$ is $0$ or
   $\bot$ [^protocol-updated-orchardkeycomponents]. If the wallet was also
   given a viewing key for the group, reject the payload when the Orchard
   component of that key is not this key.
5. If the envelope carried a group commitment item, let $\Phi_0$ be the point that
   the 32 bytes of $\mathsf{ak}$ encode, compute the member's own verifying
   share $Y_i$ from $\Phi_0$ and the item (see [Group commitment]), where $i$
   is the member's identifier, and reject the payload if the signing share
   multiplied by $\mathcal{G}^{\mathsf{Orchard}}$ is not $Y_i$. Rebuild the
   member's key material from the identifier, the signing share, $t$ and
   $\mathsf{ak}$. If the wallet knows the group's verifying shares, it SHOULD
   check that the member's verifying share matches.
6. Before it shows an address to receive funds, the wallet SHOULD check the
   viewing key against a source the member trusts, such as an address of the
   group recorded before, or the same key from at least two other members.

For key source $\mathtt{0x02}$, the wallet MUST say that the payload holds no
$\mathsf{sk}$, so it can't recover the group's notes under ZIP 2005.

A wallet SHOULD say which network a payload is for when step 2 rejects it.
The wallet scans the chain of its network from the birthday height, or from
the activation of the Orchard protocol when the birthday height is 0.

The Bech32m checksum only catches accidental changes. A string altered and
encoded again passes it, which is why step 6 applies to every key source. The
encoding of $\mathsf{ak}$ with a $\tilde{y}$ bit of $1$, which a writer
produces when it skips the negation of [Payload, version 1], is not a
canonical x-coordinate, so step 4 rejects it.

After step 4 the wallet has a valid Orchard full viewing key, whatever the
key source. The check of the group commitment in step 5 is the only check of
the signing share against the group that needs nothing outside the payload,
and it needs the group commitment item.

The fields of the payload are mandatory, and the items are not. A wallet
that does not support the value of a field of the payload MUST reject it. This covers a key source, an identifier form, and an identifier that
the wallet cannot use, such as one that is not an integer from 1 to $n$. The
wallet MUST NOT restore the member with another value in place of the one it
does not support. A wallet that does not know an item skips it (see [Items]),
and restores the member from the payload and the other items.

This ZIP leaves the following to the wallet. A wallet that is stricter than
the steps above rejects payloads that this ZIP allows, and restores no payload
that this ZIP rejects:

- whether it restores a payload that came without a group commitment item,
  whose signing share step 5 does not check against a commitment;
- which forms of a payload it reads: the payload bytes, a member backup
  string, an envelope, a backup file, a payload kept in two parts (see
  [Keeping the secret fields apart]), or a container of its own;
- what it requires of the registered and private-use items that it knows,
  under their own specification, including how their content has to agree
  with the payload.

What [Restore] says about the derivation to show the user, about retaining
$\mathsf{sk}$, and about signing nonces applies to version 2 as well. A
payload of key source $\mathtt{0x02}$ has no $\mathsf{sk}$ to retain, and the
notes of its group are not quantum recoverable.

Without a group commitment item, the restore checks the signing share only
against verifying shares that the wallet already knows, as in version 1. With
one, step 5 catches a share that does not match the commitment stored with
it: a share stored with the wrong sign, a share of another member, or a
commitment taken from before the sign of the group verifying key was fixed
(see [Group commitment]). It does not prove that the stored commitment is the
group's (see [Security Considerations]).

## Export

A wallet that exports a member backup string SHOULD export the group's unified
full viewing key with it, or tell the user that the restore needs it. ZIP 2005
asks FROST members to retain the group verifying key $\mathsf{ak}$
[^protocol-updated-orchardkeycomponents]; in version 1 it is only held by the
viewing key. A version 2 payload holds $\mathsf{ak}$ itself, and its restore
needs nothing else.

A wallet MUST NOT put signing nonces, or any other secret beyond the fields
above, in a version 1 string. A wallet MUST NOT put signing nonces in a
payload of any version, or in an item.

A wallet that exports a version 2 payload in an envelope, or in a backup
file, SHOULD include the group commitment item when it knows the group commitment. A wallet MUST NOT
export an envelope whose group commitment item and payload's signing share
fail step 5 of
[Restore, version 2].

## Storage and transmission

Wallets MUST protect every member backup string at least as securely and
reliably as $\mathsf{ask}$, as ZIP 2005 requires of $\mathsf{qsk}$
[^zip-2005-usagewithfrost], and not merely like one signing share. The same
holds for an unencrypted payload in any other form, for a backup mnemonic,
and for an encryption key. A backup file gives nothing without its encryption
key. A wallet MUST NOT write a backup file to a store, such as a cloud drive,
that also holds the encryption key that opens it, the backup mnemonic, or the
wallet seed that the key is derived from.

A recipient is a public key, but anyone who holds it can write a backup file
that its encryption key opens (see [Security Considerations]). A wallet that
keeps the recipient of an encryption key, to write backup files later, MUST
protect it as it protects the signing share, and MUST NOT disclose it.

Wallets SHOULD NOT store or transmit a string encrypted only under public-key
encryption based on discrete logarithms or on factoring, such as Orchard note
encryption, X25519, or RSA and elliptic-curve OpenPGP keys. A backup file
meets this rule: the stanzas of type `mlkem768x25519` combine X25519 with
ML-KEM-768, which is based on neither.

### Keeping the secret fields apart

A wallet does not have to keep a backup in one place. This section describes
one way to keep it in two. It is a way to store a payload and its items, which
is the wallet's choice (see [Encoding]), and its rules bind only a wallet that
offers it. It defines how the remaining part is written (see
[Format of the remaining part]), so that a split made by one wallet is put
together again by another.

The section applies to a version 2 payload whose key source is
$\mathtt{0x00}$ or $\mathtt{0x01}$, with the items that an envelope would
carry after it. Among the fields of the payload and the group commitment
item, two are secret: the signing share and $\mathsf{sk}$,
of 32 bytes each. The others are not secret: the format version, the key
source, the network, the threshold, the group size, the identifier form and
the member identifier, the birthday height, $\mathsf{ak}$ and the group
commitment.

In this section:

- the *secret part* of a payload is its signing share and its $\mathsf{sk}$,
  32 bytes each, as they are encoded in the payload;
- the *remaining part* is every other byte of the payload, in the order that
  the payload has them, followed by the items encoded as in [Items], written
  as [Format of the remaining part] specifies.

A user keeps the secret part on a medium that lasts and that the user guards,
such as a steel plate. The user keeps the remaining part on a medium that is
easier to copy and less guarded: paper, a printed QR code, a cloud drive. To
restore, the wallet takes both parts, puts the payload and its items together
again, and decodes and restores them as the contents of an envelope (see
[Format of the remaining part], [Decoding] and [Restore, version 2]).

Each secret is 32 bytes, which is what a BIP 39 mnemonic of 24 words encodes:
256 bits, followed by a checksum of 8 bits [^bip-0039]. Two such mnemonics
hold the secret part. BIP 39 is used here only to write 32 bytes as words.
The 32 bytes are the field itself. Nothing is derived from the words: the
seed derivation of BIP 39 (PBKDF2) is not applied, and neither is any
hierarchical derivation. The checksum detects an error and does not correct
it, and it is short: a mnemonic changed at random still passes with
probability 1 in 256.

The signing share differs for each member. $\mathsf{sk}$ is the same for every
member of the group, so every member of a group has the same second mnemonic.

A wallet that offers this split:

- MUST put together, before it decodes, exactly the bytes of the payload and
  the items that were split;
- MUST NOT put the signing share or $\mathsf{sk}$ in the remaining part;
- MUST NOT put in the remaining part, in the clear, an item whose
  specification says that its value is a secret. A registered item can hold
  data whose sensitivity its registrant's specification states;
- MUST use the 32 bytes of a field as the entropy of its mnemonic, when it
  writes a secret as a BIP 39 mnemonic, and MUST NOT apply the seed
  derivation of BIP 39 to that mnemonic;
- SHOULD tell the user which mnemonic is the signing share and which is
  $\mathsf{sk}$, and that neither is the seed of a wallet;
- SHOULD keep the group commitment item in the remaining part, so that step 5 of
  [Restore, version 2] checks a signing share that was copied by hand;
- MUST write and read the remaining part as [Format of the remaining part]
  specifies;
- SHOULD encrypt the remaining part under a key derived from the secret part,
  that is write it with mode $\mathtt{0x01}$ of
  [Format of the remaining part];
- SHOULD tell the user that the secret part does not restore the member
  without the remaining part.

The first rule of [Storage and transmission] applies to the secret part, in
whatever form it is written, as it does to a whole payload. It does not apply
to the remaining part. The rest of this section says what each part gives to
a party that holds it alone.

**What the remaining part gives.** A party that finds the remaining part,
without the secret part, learns the following.

| Fields | What their finder learns |
|--------|--------------------------|
| format version, key source | that the bytes are a FROST member backup, and which derivation the group uses |
| network, threshold $t$, group size $n$, member identifier | that the holder is member $i$ of a $t$-of-$n$ group on that network |
| birthday height | a height below which the group has no transaction, so roughly when the group was created |
| $\mathsf{ak}$ and the group commitment | the group's Spend validating key, and the verifying share of every member (see [Group commitment]) |

With these fields alone the finder learns no address, no balance and no
transaction of the group. The group's Orchard full viewing key is
$\mathsf{ak}$, $\mathsf{nk}$ and $\mathsf{rivk}$. Both derivations of
[Derivations] compute $\mathsf{nk}$ and $\mathsf{rivk}$ from $\mathsf{sk}$,
which is not in the remaining part. The incoming viewing key, the outgoing
viewing key and every address are derived from all three values
[^protocol-updated-orchardkeycomponents], so the finder has none of them.
On the chain, a spend of the group shows a key $\mathsf{rk}$ that is
$\mathsf{ak}$ re-randomized, which the protocol specification does so that
the key shown cannot be linked to the key it was made from
[^protocol-spendauthsig]. The finder does not hold the randomizer of any
spend, so $\mathsf{ak}$ does not tell it which transactions are the group's.
$\mathsf{ak}$ does let the finder recognise the group's full viewing key, or
another member's payload, if it sees one.

The finder cannot sign, and so cannot spend: it holds no signing share. It
cannot restore the member, since [Restore, version 2] needs the signing share
and $\mathsf{sk}$. It cannot view the group's transactions, as said above.

$\mathsf{ak}$ and the verifying shares are not secrets, but they are not
values to publish either. Each is the public key of a secret scalar:
$\mathsf{ak}$ of the group's Spend authorizing key, and a verifying share of
a member's signing share. An adversary that can compute discrete logarithms
on the Pallas curve, for instance with a quantum computer, computes from the
remaining part the Spend authorizing key and every member's signing share.
What it can then do depends on the protocol in force:

- While the current Orchard protocol is enabled, ZIP 2005 states that such an
  adversary can break spend authentication for any shielded address, known
  to it or not, after finding a single discrete logarithm
  [^zip-2005-dlog-before]. The remaining part is not what makes that attack
  possible.
- Under the Recovery Protocol of ZIP 2005, a spend is assumed to need both a
  proof of knowledge of $\mathsf{qsk}$ and a signature that verifies under
  $\mathsf{ak}$ [^zip-2005-threat-model]. With the Spend authorizing key the
  adversary makes the signature. It does not hold $\mathsf{qsk}$, which is
  derived from $\mathsf{sk}$, so the remaining part alone does not let it
  spend through the Recovery Protocol. It lacks only what every member
  holds: one leaked $\mathsf{sk}$, from any member, completes it.

ZIP 2005 already warns that "a quantum adversary may be able to steal the
funds with only access to $\mathsf{qsk}$" [^zip-2005-usagewithfrost] (see
[Security Considerations]). It does not say from which public values such an
adversary would compute the Spend authorizing key, and this ZIP does not
establish whether an adversary that holds $\mathsf{qsk}$ and has never seen
$\mathsf{ak}$ could do so. An adversary that holds the remaining part has
$\mathsf{ak}$ and does not need to look for it. This applies to a group of
the ZIP 2005 derivation. The notes of a legacy group are not quantum
recoverable (see [Derivations]).

**What a party that can change the remaining part can do.** The remaining
part in the clear is not authenticated. A party that reads it knows the
member's verifying share $Y_i$. It can choose another $\mathsf{ak}$, for
instance one whose secret it holds alone, and compute a group commitment
that still gives $Y_i$ for the member's identifier. A payload put together
from the unchanged secret part and this changed remaining part passes every
step of [Restore, version 2], step 5 included. The wallet then shows
addresses of a viewing key built around that other $\mathsf{ak}$. The group
cannot spend what these addresses receive, since its signatures do not verify
under that $\mathsf{ak}$. Whether the party can spend it depends on
$\mathsf{sk}$, from which $\mathsf{nk}$ and $\mathsf{rivk}$ are derived: a
party that also holds $\mathsf{sk}$, such as another member, can, and this is
the attack that [Security Considerations] describes for a viewing key
obtained elsewhere. A party without $\mathsf{sk}$ cannot, and the funds are
then out of reach of both. The encryption described below prevents the
change by any party that does not hold both this member's signing share and
$\mathsf{sk}$: another member holds $\mathsf{sk}$ but not this share. Without
it, a wallet SHOULD check the restored viewing key against a source the member
trusts, as [Restore] asks, before it shows an address.

**What the secret part gives.** A party that finds the secret part, without
the remaining part, holds one signing share and $\mathsf{sk}$.

- The signing share alone signs nothing: fewer than $t$ shares cannot spend
  (see [Security Considerations]).
- $\mathsf{sk}$ gives $\mathsf{nk}$ and, under the ZIP 2005 derivation,
  $\mathsf{qsk}$ and $\mathsf{qk}$. It does not give the full viewing key,
  because $\mathsf{ak}$ is missing. Under the ZIP 2005 derivation
  $\mathsf{rivk}$ is computed from $\mathsf{ak}$ as well, and under both
  derivations the incoming viewing key and the addresses are (see
  [Derivations] and [^protocol-updated-orchardkeycomponents]). So the finder
  of the secret part alone views nothing.
- $\mathsf{ak}$ is not a secret. It is in the remaining part, in the group's
  unified full viewing key and in every other member's payload. A party that
  holds $\mathsf{sk}$ and obtains $\mathsf{ak}$ from any of these holds the
  group's full viewing key and sees every transaction of the group.
- $\mathsf{sk}$ gives $\mathsf{qsk}$, with what [Security Considerations]
  says of a leaked $\mathsf{qsk}$.

The secret part is therefore still to be protected as [Storage and
transmission] requires of a whole payload. What the split changes is that the
remaining part can be kept with less care.

A mnemonic copied with an error can still pass its checksum. A wrong signing
share then fails step 5 of [Restore, version 2] when the remaining part holds
a group commitment item. A wrong $\mathsf{sk}$ fails no step: the wallet derives
another viewing key and shows another address.

**Encrypting the remaining part.** A wallet that offers the split SHOULD
encrypt the remaining part under a key derived from the secret part: a hash of
the signing share followed by $\mathsf{sk}$, with a domain separation of its
own, as the key of an authenticated cipher (see
[Format of the remaining part]). A wrong $\mathsf{sk}$ fails no step of
[Restore, version 2], and this encryption is what detects it. It costs the
user nothing, because whoever restores the member holds both secrets and needs
no other code. It gives the following:

- the remaining part found alone is opaque: its finder learns none of the
  fields above, and an adversary that computes discrete logarithms finds no
  public key in it. The cipher is symmetric, so recording the ciphertext
  today gives nothing later;
- the remaining part is authenticated: a changed one does not decrypt, and
  only a party that holds both secrets of this member can write one that
  decrypts;
- a secret copied with an error is detected, the signing share and
  $\mathsf{sk}$ alike, and so are the two secrets given in the wrong order and
  the secret part of another member: the remaining part does not decrypt.

It does not protect the secret part. The secret part is the key: whoever holds
it opens the remaining part, and also writes one that the wallet accepts. The
secret part is what the user keeps apart and guards.

This is not a backup file (see [Backup file]). A backup file encrypts a
whole payload, its secrets included, under an encryption key that comes from
a wallet seed or a backup mnemonic. Here the key comes from the secrets that
the user keeps for the member, and the secrets are not in what is
encrypted.

**If the remaining part is lost.** The secret part alone does not restore the
member. The network, the threshold, the group size, $\mathsf{ak}$ and the
group commitment are the same in every member's payload, so any other member
can supply them. Only the identifier is the member's own. With the group
commitment and the signing share, an identifier of the short form is found
again: it is the $i$ from 1 to 255 for which the signing share multiplied by
$\mathcal{G}^{\mathsf{Orchard}}$ equals $Y_i$. An identifier of the full form
is not found this way; it is not secret, and a user can keep it next to the
secret part. The birthday height can be replaced by 0. An item that another
member does not hold is not recovered.

Values supplied by another member are trusted as much as a viewing key
obtained from another member: a dishonest member can supply an $\mathsf{ak}$
and a group commitment of its own making, as described above. The checks that
[Restore] asks for before showing an address apply.

#### Format of the remaining part

The *plaintext* of a remaining part is the following bytes, where $\ell$ is
the length of the member identifier field, 1 in the short form and 32 in the
full form:

| Field | Size | Offset, short | Offset, full |
|-------|------|---------------|--------------|
| format version, $\mathtt{0x02}$ | 1 | 0 | 0 |
| key source, $\mathtt{0x00}$ or $\mathtt{0x01}$ | 1 | 1 | 1 |
| network | 1 | 2 | 2 |
| threshold $t$ | 1 | 3 | 3 |
| group size $n$ | 1 | 4 | 4 |
| identifier form | 1 | 5 | 5 |
| identifier | $\ell$ | 6 | 6 |
| birthday height | 4 | 7 | 38 |
| $\mathsf{ak}$ | 32 | 11 | 42 |
| items | rest | 43 | 74 |

These are the fields of the payload as [Payload, version 2] defines them, in
the same order, without the signing share and $\mathsf{sk}$. The payload part
of the plaintext is $42 + \ell$ bytes: 43 in the short form, 74 in the full
form. The items follow it, encoded as in [Items], and take all the remaining
bytes; there can be none.

A remaining part is the following bytes:

| Size   | Field |
|--------|-------|
| 4      | magic: $\mathtt{0x8A}$ $\mathtt{0x5A}$ $\mathtt{0x46}$ $\mathtt{0x52}$ ($\mathtt{0x8A}$, then `ZFR` in ASCII) |
| 1      | container version, $\mathtt{0x01}$ |
| 1      | mode |
| varies | mode parameters, fixed by the mode |
| rest   | body |

| Mode | Meaning | Mode parameters | Body |
|------|---------|-----------------|------|
| $\mathtt{0x00}$ | in the clear | none | the plaintext |
| $\mathtt{0x01}$ | encrypted under the secret part | salt (32 bytes), then nonce (24 bytes) | the encrypted plaintext, 16 bytes longer than the plaintext |
| other | unassigned | | |

Under mode $\mathtt{0x01}$, the plaintext is encrypted with XChaCha20-Poly1305
[^xchacha], with the following values:

salt

: 32 bytes, stored in the clear. An encoder MUST draw them from a
  cryptographic random number generator for each remaining part it writes.

nonce

: 24 bytes. An encoder MUST draw them from a cryptographic random number
  generator for each remaining part it writes.

key

: $\mathsf{BLAKE2b\text{-}256}(\texttt{"ZcashFROSTRemain"},\, \mathsf{share} \,\|\, \mathsf{sk} \,\|\, \mathsf{salt})$,
  that is BLAKE2b [^protocol-concreteblake2] with a 32-byte output, the
  16-byte personalization `ZcashFROSTRemain`, no key and no salt parameter,
  applied to the 96 bytes $\mathsf{share} \,\|\, \mathsf{sk} \,\|\, \mathsf{salt}$,
  where $\mathsf{share}$ and $\mathsf{sk}$ are the 32-byte fields of the secret
  part as the payload encodes them.

associated data

: The 38 bytes of the remaining part that precede the nonce: the magic, the
  container version, the mode and the salt.

body

: The XChaCha20-Poly1305 encryption of the plaintext under the key, the nonce
  and the associated data: the ciphertext, as long as the plaintext, followed
  by the 16-byte tag.

The personalization differs from that of the encryption key of a backup
mnemonic (see [Encryption key from a backup mnemonic]), so a key derived for a
remaining part is never the encryption key of a backup file.

A remaining part written as text follows the rules of the text of an envelope
(see [Envelope]): an encoder writes the base64url encoding of its bytes without
padding, and a reader ignores whitespace and accepts either base64 alphabet,
with or without padding. Its text starts with `ilpGUgE` (`ilpGUgEA` under mode
$\mathtt{0x00}$, `ilpGUgEB` under mode $\mathtt{0x01}$). Its first byte,
$\mathtt{0x8A}$, is not the first byte of a payload or of a member backup
string, and its fourth byte, $\mathtt{0x52}$, is not that of an envelope
($\mathtt{0x42}$), so a reader tells it from each of these by its first bytes.

The input of a reader is a remaining part, and the signing share and
$\mathsf{sk}$ of a secret part. A reader MUST reject the remaining part if any
of the following holds:

- it does not start with the four magic bytes above;
- it is shorter than 6 bytes, so that it has no container version or no mode;
- the container version is not $\mathtt{0x01}$;
- the mode is neither $\mathtt{0x00}$ nor $\mathtt{0x01}$;
- the mode is $\mathtt{0x01}$, and the remaining part is shorter than 78
  bytes or its body does not decrypt with the key derived from the given
  signing share and $\mathsf{sk}$, and the nonce and associated data above;
- the plaintext is shorter than 6 bytes, or its format version is not
  $\mathtt{0x02}$, its key source is neither $\mathtt{0x00}$ nor
  $\mathtt{0x01}$, or its identifier form is neither $\mathtt{0x00}$ nor
  $\mathtt{0x01}$;
- the plaintext is shorter than $42 + \ell$ bytes.

The plaintext is the body under mode $\mathtt{0x00}$, and the decrypted body
under mode $\mathtt{0x01}$. The reader then puts the payload and its items
together again. They are, in this order:

1. the first $6 + \ell$ bytes of the plaintext, up to and including the
   member identifier;
2. the signing share of the secret part;
3. the next 4 bytes of the plaintext, the birthday height;
4. $\mathsf{sk}$ of the secret part;
5. the next 32 bytes of the plaintext, $\mathsf{ak}$;
6. the rest of the plaintext, the items.

These bytes are the contents of an envelope: the wallet decodes them as
[Decoding] specifies for the contents of an envelope, and restores the member
under [Restore, version 2]. A payload or an item that fails there is rejected
as it would be in an envelope.

Under mode $\mathtt{0x01}$, a body that does not decrypt means that the secret
part given is not the one that the remaining part was written with, or that
the remaining part was changed or damaged. A signing share or an $\mathsf{sk}$
copied with an error, the two given in the wrong order, and the secret part of
another member of the group all give another key. The reader cannot tell
these cases apart, and restores nothing. Under mode $\mathtt{0x00}$ nothing
checks the remaining part or the secret part against each other before
[Restore, version 2], which catches a wrong signing share at step 5 when the
remaining part holds a group commitment item, and does not catch a wrong
$\mathsf{sk}$.


# Rationale

**Bech32m.** It gives a checksum that catches copy errors and a string users
can paste or show as a QR code. An uppercase string fits the QR alphanumeric
mode, which makes the code smaller. Its checksum guarantees the detection of
any error affecting at most 4 characters only for strings of at most 90
characters [^zip-0173] [^bip-0350]. A version 1 string is 123 characters
long, so it keeps the 30-bit checksum without that guarantee, as unified
addresses and viewing keys do [^zip-0316].

**No network in a version 1 string.** A member's share is the same on every
network the group lives on. The viewing key carries the network. Version 2
names the network because it holds a birthday height, and a height means
something on one network only. The network is a byte of the payload and not
in the human-readable part: decoders check the prefix before the version, so
a version 1 decoder given another prefix would report a wrong prefix instead
of an unsupported version. A restore rejects a payload of another network
rather than moving it, so that a test backup is never restored on Mainnet.
Regtest has no value, since ZIPs do not specify Regtest.

**$\mathsf{ak}$ is not stored in version 1.** The restore needs the viewing
key anyway to scan the chain, and checking $\mathsf{nk}$ and $\mathsf{rivk}$
against it catches a string from another group. The cost is that the string
does not authenticate $\mathsf{ak}$ (see [Security Considerations]). Version 2
stores $\mathsf{ak}$, or the whole Orchard full viewing key for a group that
has no $\mathsf{sk}$, so that nothing outside the payload is left to mismatch.
It stores the raw key and not its unified encoding, which would repeat the
network.

**$\mathsf{sk}$ is stored whole.** ZIP 2005 makes it necessary "to retain, or
be able to rederive, $\mathsf{sk}$" [^protocol-updated-orchardkeycomponents],
and every member already holds it, along with $\mathsf{qsk}$
[^zip-2005-usagewithfrost].

**In version 1, the derivation is found, not recorded.** Version 1 is Zkool's
existing payload, which has no field for it. With two derivations and the
viewing key at hand, trying both is cheap and unambiguous: both matching would
need a collision in the hash functions behind $\mathsf{rivk}$. Version 2
records the derivation in the key source, so a restore tries nothing, and a
third derivation can be added as a new key source without ambiguity. The key
source $\mathtt{0x02}$ covers groups that kept no $\mathsf{sk}$, such as those
made by zcash-sign's `generate`, which Konclave uses.

**One-byte identifier in version 1.** It covers groups whose identifiers are
the integers 1 to $n$, as in Zkool and in frost-client's trusted-dealer key
generation. frost-client's DKG and Zafe derive 32-byte identifiers, which
version 2 holds in its full form. Version 2 requires the short form whenever
it fits, so that an identifier has one encoding, and does not compare the
identifier with $n$, so that a group refreshed to remove a member still fits.

**A threshold of at least 2.** With $t = 1$ the polynomial of key generation
has degree 0: it is the constant $f(0)$, so every member's signing share is
the group's Spend authorizing key itself. Each member spends alone, a backup
of one member is a backup of the whole key, and the group commitment item
would be empty. Such a key is a single-signer key held in several copies, and
a format for threshold members has nothing to add to it. Both versions
therefore reject $t < 2$ at decoding.

**Keys are checked at restore, not at decoding.** Version 1 checks
$\mathsf{sk}$, the signing share and the derived keys in its restore steps,
and its decoding checks only the layout and the bounds of $t$, $n$ and $i$.
Version 2 keeps this split, as the text discussed in zcash/zips#1413 does:
the identifier checks are decoding checks, and the canonical share is a
restore check. Decoding needs no curve arithmetic beyond reading scalars and
the points of the group commitment, and the checks that give the payload its
meaning are in one list, in [Restore, version 2].

**A signing share of zero is rejected.** Zero is the canonical encoding of a
Pallas scalar, but it is no share: its verifying share is the zero point,
which the element encoding of ZIP 312 does not accept [^zip-0312]. A member
restored from it could never sign. Both versions reject it at the restore step
that checks the canonicity of the share. Neither version 1 as first published
nor the version 2 text discussed in zcash/zips#1413 names the case, so the
rule is added here.

**One sign for everything stored.** Orchard requires an $\mathsf{ak}$ whose
encoding has a $\tilde{y}$ bit of $0$, and key generation meets this by
negating the whole key material when needed. A payload holds only values from
after that step. A restore then never negates anything, and $\Phi_0$ does not
need to be stored, since $\mathsf{ak}$ is its encoding. Storing the points
that the participants of a distributed key generation exchanged would mean a
second form of the same group, and a negation to apply again at every
restore, which is the step that is easy to forget.

**The payload and the envelope.** The version 2 payload is the one discussed
in zcash/zips#1413 [^zips-1413-v2], unchanged, so every decoder and every test
vector written against that text holds for this ZIP. What this ZIP adds is
around the payload: the envelope, and the items it carries. Wallets differ in
where a backup lives: a file, a QR code, a store that has its own encryption.
A rule that names one of these would bind every wallet and make no payload
easier to restore, so this ZIP has none. The string and the envelope are
defined so that the wallets which want text, or extra data and encryption,
do it in one way.

**A payload kept in two parts.** A user who engraves a backup on steel wants
to engrave little. Only the signing share and $\mathsf{sk}$ have to be kept as
secrets, and each has the size that a mnemonic of 24 words holds. The rest of
a payload and of its items is not secret, and every member holds almost the
same. [Keeping the secret fields apart] says how a wallet keeps the two
secrets apart from the rest, how the rest is written, and what a party learns
from each part.

**The remaining part encrypted under the secret part.** The key of the
remaining part is derived from the two secrets and not from a code of its
own. A separate code would be a third secret for the user to keep, and losing
it would lose the remaining part. The secrets are already kept, and whoever
restores holds them. Deriving the key from both secrets also makes the
encryption a check of the secret part: an $\mathsf{sk}$ copied with an error
passes every step of [Restore, version 2], and fails here. The remaining part
has its own magic rather than a wrapper id of the envelope, because a
remaining part is read only with a secret part and holds no payload until the
two are put together. The salt is not needed for the key to be unique, since
the signing share differs for each member; it makes the key differ for each
remaining part that is written.

**Telling the forms apart.** A decoder can be given five kinds of input: a
string, which starts with `zkfrost1`; the bytes of an envelope, whose first
byte $\mathtt{0x8A}$ is not printable ASCII and not a Bech32m character, so
that no UTF-8 text starts with it; the text of an envelope, which starts
with `ilpGQgE`; a backup file, which starts with `age-encryption.org/v1`; and
the text form of a backup file, which starts with `-----BEGIN`. The bytes of a
payload start with $\mathtt{0x01}$ or $\mathtt{0x02}$. A decoder therefore
tells them apart from the first bytes, without being told which one it was
given. The text of an envelope needs no prefix of its own: the magic already
identifies it. A remaining part is not another kind of input of a decoder,
since it is read only with a secret part, but it is told apart the same way:
its magic, $\mathtt{0x8A}$ then `ZFR`, differs from the envelope's in its
fourth byte, so its text starts with `ilpGUgE` and not `ilpGQgE`.

**One wrapper.** An unencrypted payload without items needs no envelope:
the payload bytes or the string carry it. Wrapper $\mathtt{0x00}$ carries a
payload with its items in the clear, for a backup file to encrypt or for a
store that has its own encryption. Encryption is left to age, so the envelope
needs no other wrapper. The wrapper id stays in the header so that a later
ZIP can define another wrapper without a new container version.

**age and not a new cipher.** age is a specified file format, and the `age`
tool, from version 1.3, opens a backup file without any Zcash software: a user
who has lost every wallet still gets the envelope back. Its post-quantum hybrid recipients meet the rule of
[Storage and transmission] on public-key encryption: a file stays secret as
long as ML-KEM-768 or X25519 does. Its recipients let a wallet write a backup
without holding the secret that opens it, for instance to write a new backup
after each change of the group, with the encryption key kept offline. A
cipher defined by this ZIP would give none of this, and would need its own
review.

**A 128-bit file key.** age draws a file key of 16 bytes, so a backup file
has the margin of a 128-bit key, whatever its encryption key: a quantum
adversary that searches for the file key with Grover's algorithm needs about
$2^{64}$ evaluations (see [Security Considerations]). An earlier draft of
this text offered a 256-bit recovery code for backups meant to last. A backup
file cannot offer that, since age fixes the length of the file key. This is
the cost of a format that the `age` tool opens.

**Two sources for the encryption key.** Most users already keep a wallet
seed, and a key derived from it adds nothing to keep. Some users hold no
seed of a ZIP 32 wallet, or do not want the backup of a group to depend on
the seed of a personal wallet; a backup mnemonic serves them. It is 128
random bits, as strong as the file key of age, so it needs no password
hashing, and opening a file stays fast on a phone. A user cannot choose it,
which is the cost of leaving password hashing out.

**ZIP 325's private-use key, under account 0.** ZIP 325 already defines a
tree of keys, derived from the wallet seed, for data that a wallet keeps
outside the chain, and reserves a private-use child in it. Using it needs no
new registered derivation. ZIP 32's hardened derivation separates the key
from every spending and viewing key of the seed. The account is fixed to 0,
so a seed gives one key whatever account the wallet uses, and a user who
restores a seed does not need to know which account made the backup.

**A marker word before the 12 words.** 12 BIP 39 words look like a seed
phrase, and a user could type them into a wallet as one. The wallet would
then create an empty wallet from them, and the user could take the backup
mnemonic for a wallet seed. The marker `zbackup` is not a BIP 39 word, so a
wallet rejects the 13 words as a seed phrase. A reader accepts the 12 words
without the marker, since a user can forget it and it holds no information.

**No metadata inside a backup file.** The `age` tool refuses a file with a
line before or after the armor, and the binary format has no place for data
that age does not define. Any such line would make the file unreadable without software of
this ZIP. Whatever a wallet wants to keep with the payload goes in the
envelope's items, inside the encryption.

**A hint in the file name.** A wallet that is given a file needs to know
which key to try: the key of its wallet seed, or a backup mnemonic to ask the
user for. Nothing in an age file says it, by design. A file name is the
one place outside the file that a user sees and that travels with the file.
The name is a hint only, because a user can rename a file.

**Items sit in the envelope.** Items are kept out of the payload for three
reasons. Nothing deployed or being built against the version 2 text of
zcash/zips#1413 changes: its payload, its seven decoding rules, its restore
steps and its test vectors stand as they are, and a decoder of that text
never sees an item. A string decoder never sees one either, since a string
holds a payload and nothing else. And long data never suited Bech32m: a group
commitment and a wallet's items can reach several kilobytes, which is
thousands of characters, while the Bech32m checksum is designed for strings
of at most 90 characters [^zip-0173] [^bip-0350], and nobody copies a string
of that length by hand. The envelope is bytes, which a file or a QR code
holds directly. The group commitment is the first item: a member restored
without help needs it to check its own signing share, and to compute the
other members' verifying shares, without asking another member (see
[Group commitment]).

**Items instead of new versions.** A new format version stops every older
wallet from restoring. Most additions do not need that: a wallet that ignores
the group commitment or another wallet's data still restores the member. An
item is skipped by a decoder that does not know it, so such data is added
without a new version. A change that an older wallet must not ignore still
takes a new format version. Strict typecode order and one item per typecode
keep one encoding per envelope. ZIP 316 treats the items of a unified address
or viewing key the same way: ascending typecode order, no typecode twice, and
unknown items ignored [^zip-0316]. The typecode is a
$\mathtt{compactSize}$ as in ZIP 316, and the private-use range stops at
$\mathtt{0xFC}$, the greatest value encoded in one byte. When the one-byte
typecodes run out, typecodes are assigned in the reserved range with no new
format version, and a decoder written before that skips their items.

**Registered and private-use typecodes.** A wallet can have data of its own
to put in a backup. Without a place for it, each wallet would wrap the payload in a format of its own, and backups
would stop being portable. A registered range gives a wallet typecodes that
no one else uses, at the cost of one line in this ZIP; the layout stays in
that wallet's specification, because no other wallet reads it. The
private-use range needs no registration and so can collide, which is why its
values start with a tag. ZIP 316 has only the second kind: it reserves six
typecodes ($\mathtt{0xFFFA}$ to $\mathtt{0xFFFF}$) for experiments made before
a ZIP assigns a typecode [^zip-0316-experimental], and assigns every other
typecode itself. It has no range registered to one wallet. This ZIP adds one
because wallet-specific data is expected to stay in a backup for good, which
an experiment is not.

**The group commitment as the group's public data.** The $t$ points of the
group commitment, one of which is $\mathsf{ak}$, determine the verifying share
of every member, for any identifier, including a member added later under
the same polynomial. Storing the verifying shares themselves takes $n$ points
and the list of identifiers, and does not let a member check that they belong
to one polynomial. The transcript of a distributed key generation proves
more, but it is several times larger, its layout depends on the key
generation protocol, and a group made by a trusted dealer has none.

**Recognising a restored member.** A payload restores key material only. How
the other members recognise the restored member, and let it back into their
exchanges, depends on the transport of the wallet, which this ZIP does not
specify. One way needs nothing beyond the payload: the restored member signs a
challenge with its signing share, and the others verify the signature against
that member's verifying share, which they compute from the group commitment.
The group commitment is kept in every backup so that this way stays open.

**Version 1 is Zkool's payload.** Backups users already made keep working,
and the first implementation exists today.


# Security Considerations

A string holds one signing share. Fewer than $t$ strings cannot spend, as long
as discrete logarithms on the Pallas curve stay hard.

Every string also holds $\mathsf{sk}$, which yields $\mathsf{qsk}$. ZIP 2005
notes that "a quantum adversary may be able to steal the funds with only
access to $\mathsf{qsk}$, which is held by every participant"
[^zip-2005-usagewithfrost]. Against such an adversary the threshold may give no
protection: one leaked string, from any member, may be enough to steal the
funds. ZIP 2005 requires each participant to "treat $\mathsf{qsk}$ as a
secret held at least as securely and reliably as $\mathsf{ask}$"
[^zip-2005-usagewithfrost], which is why [Storage and transmission] asks the
same for every string.

For the same reason, a string that is stored or sent encrypted only under
public-key encryption based on discrete logarithms or on factoring is exposed
to an adversary who records the ciphertext now and gains a quantum computer
later. A backup file does not have this weakness, because its stanzas of
type `mlkem768x25519` include ML-KEM-768: the file key stays secret as long as
ML-KEM-768 or X25519 does. A passphrase with a memory-hard KDF and an AEAD
(see [Open issues]) would not have it either.

Whoever holds a backup file and its encryption key holds the payload. The
file key of age is 128 bits, so a backup file is at most as strong as a
128-bit key, whatever secret its encryption key comes from. A backup mnemonic
has 128 bits of entropy. A mnemonic that is not random, for instance one
derived from a passphrase, would be open to guessing, since the derivation of
the encryption key is fast. A quantum adversary that holds a backup file can
search for its file key with Grover's algorithm, in about $2^{64}$
evaluations, the margin of AES-128.

An encryption key derived from a wallet seed is a leaf of ZIP 32's hardened
derivation. A party that learns it learns nothing about the spending keys or
the viewing keys of that seed. The seed, however, gives the encryption key:
whoever holds the wallet seed and a backup file of that seed holds the
payload, so a wallet seed also protects the signing share and $\mathsf{sk}$ of every member
whose backup file it opens. A party that learns the seed of a wallet also
opens every backup file written for it, on the same network.

Only a string has a checksum. The payload bytes have none, and neither has an
envelope under wrapper $\mathtt{0x00}$. A backup file detects damage: a
changed byte makes it fail to open. Damaged bytes can decode to a wrong
share, which the group commitment item catches when the envelope carries one (see
[Restore, version 2]). A wallet that stores an unencrypted payload outside a
string has to protect its integrity itself. A remaining part in the clear has
no checksum either.

A remaining part encrypted under the secret part is as hard to open or to
forge as the secret part is to guess: the key is derived from a signing share
and $\mathsf{sk}$, 64 bytes, of which the signing share is known only to the
member. Every other member holds $\mathsf{sk}$, so against them the key rests
on the signing share alone. The encryption gives the secret part itself no
protection: a party that holds it opens the remaining part, writes a new one,
and holds what [Storage and transmission] says a payload gives.

The group commitment item is not authenticated by anything outside the
payload. The check in [Restore, version 2] catches a share and a commitment
that do not match, which is what damage or a wrong sign produce. It does not
catch a payload whose share and commitment were both replaced. A member that
restores from a payload it did not keep itself has to compare the verifying
shares it computes with those the other members hold.

The same holds for every other field. A version 2 payload and its items carry
no proof that the $\mathsf{sk}$, the $\mathsf{ak}$, the group commitment and
the other items come from one key generation: whoever can write them can
write a payload and items that pass every step of [Restore, version 2] for a
group of their own making. An unencrypted payload, in any form, can be
written by anyone.

A backup file does not authenticate its writer either. Whoever holds a
recipient can write a backup file that the matching encryption key opens,
with a payload and items of their choice. A party that can also replace the
files a user keeps, for instance on a cloud drive, can make the user restore
a group of the party's making, and show addresses whose funds the party can
spend. This is why [Storage and transmission] asks a wallet to protect a
recipient as it protects the signing share and not to disclose it. The checks
of [Restore] and [Restore, version 2] apply to the contents of a backup file
as to any other payload, including the check of the viewing key against a
source the member trusts before an address is shown.

A backup made before a group's shares were replaced, by a refresh or a
resharing, holds a share that no longer signs with the others. This ZIP does
not detect it. The group does, when the restored member's share fails to
verify.

The restore does not prove the signing share (see [Restore]). A wrong share,
for instance one exported with the wrong sign (see [Payload, version 1]),
gives signature shares that fail verification. It stops that member from
signing until it is restored from a correct string, but it cannot sign
anything or leak the other shares.

The restore does not authenticate $\mathsf{ak}$ either. Every member holds
$\mathsf{sk}$, so a dishonest member can build a viewing key around another
$\mathsf{ak}$, for instance one whose secret it holds alone, with the
$\mathsf{nk}$ and $\mathsf{rivk}$ that either derivation gives for it. A string
restored with that key passes every check in [Restore], and the wallet then
shows addresses that only that member can spend from. The wallet would not see
the group's notes, and the group's signatures would not verify under that
$\mathsf{ak}$, but funds received at those addresses before anyone notices are
lost to the group. Checking the viewing key against a trusted source, as
[Restore] asks, prevents this. A version 2 payload carries $\mathsf{ak}$, so
its restore takes $\mathsf{ak}$ from the member's own backup and not from a
viewing key obtained elsewhere.

A member restored from a string has the same authority as before: the string
adds no signer to the group. It carries no nonces, so a restore cannot bring
back a nonce that was already used or published.


# Deployment

This ZIP changes no consensus rule. Wallets can adopt it at any time.


# Reference implementation

- Zkool: `rust/src/frost/backup.rs` in hhanh00/zkool2, released in Zkool
  6.35.0 [^zkool-backup].
- frost-zec: `src/backup.rs` in OpenPGP-Wallet/frost-zec, by one of this
  ZIP's owners, written from this ZIP's rules apart from Zkool's code, with
  `backup export` and `backup restore` commands [^frost-zec-backup].

Both implement version 1.

# Test vectors

This section has the test vectors of version 1, then two of version 2
(see [Test vectors for version 2]), then those of the backup file (see
[Test vectors for the backup file]).

The version 1 test vectors are generated by the Python code proposed in
zcash/zcash-test-vectors#136 [^zcash-test-vectors-pr], in three files:
`frost_member_backup`, `frost_member_backup_decoding` and
`frost_member_backup_restore`.
They cover a 2-of-3 ZIP 2005 group, a 2-of-3 legacy group together with the
ZIP 2005 viewing key built from the same $\mathsf{sk}$ and $\mathsf{ak}$ (whose
strings restore under either key, with a different derivation), and a 3-of-5
ZIP 2005 group; an uppercase string; seventeen strings that decoders must
reject, two of them for the padding rule and one with a threshold of 1; and
three cases that decode but that the restore must reject: a valid string
paired with another group's viewing key, a string whose signing share is not
canonical, and a viewing key with no Orchard component.

One of the seventeen strings predates version 2 and has to be replaced (see
[Open issues]). It is a valid version 1 payload whose first byte was changed
to $\mathtt{0x02}$, to be rejected for its format version:

    zkfrost1qgqsyqev42j6ml9d3tqthrzsmndmc854fqjmuspljl6yhswkc4p7zh208vjv7y78y36c3psy3czcxn2apfjya8ywz4whuzav8uj4s6mv4gguxtl2y2v

A decoder of this ZIP reads it as a version 2 payload. It still rejects it,
but at rule 4 of [Decoding], for the byte $\mathtt{0x02}$ where a version 2
payload has its network and the byte $\mathtt{0xAA}$ where it has its
identifier form.

Every valid string decodes to the same fields and matches the stated
derivation with Zkool's code, and every $t$-subset of members restored from
their strings produces a FROST signature that verifies under $\mathsf{ak}$.
Zkool 6.35.0 rejects every invalid string except three. It does not check
the padding bits, and decodes the two padding strings to the same fields as
the valid string they come from; it also accepts a threshold of 1. Zkool
6.36.0 rejects all three: it has the padding fix, hhanh00/zkool2#1468
[^zkool-padding-fix], and its decoder now refuses a threshold below 2.
A third decoder, by the other owner of this ZIP, with the derivations written
directly from BIP 350, ZIP 316 and ZIP 2005 and no Zcash library, gives the
expected result for every string, both viewing keys of each group, and every
rejected case. With frost-client's version of `reddsa`, every $t$-subset also
rebuilds $\mathsf{ak}$ and signs.

One member of the 2-of-3 ZIP 2005 group (byte strings in hex):

    sk      24cf13c724758886048e05834d5d0a644e9c8e155d7e0bac3f25586b6caa11c3
    ak      a0dffa5f21db1b2c6cc72d9b6af9d777884c2c7d3e06ad479732bec46cee110d
    nk      e32868e0d0770d63ad17b8a807c8133bfb957c81fb973f1c43531c7cc15e5831
    qsk     7193049fb014e408fd139cfcc47c58d6c20451473bab44ccb9fffc983ecf8ada
    qk      eec37a6202cd2f8b1bea8279bf87ef0dcb2bf73bbec28f077c4c5e1a743dd79f
    rivk    d0971e49314b6ad96d67af2766afc7db08fe6bf01f3ab0d84376016aad6ef419
    t, n    2, 3
    id      1
    share   2caaa5adfcad8ac0bb8c50dcdbbc1e954825be403f97f44bc1d6c543e15d4f3b
    string  zkfrost1qyqsyqev42j6ml9d3tqthrzsmndmc854fqjmuspljl6yhswkc4p7zh208vjv7y78y36c3psy3czcxn2apfjya8ywz4whuzav8uj4s6mv4gguxm4ktp7
    ufvk    uview1mvknnjr2s9cuc7z5kjsd3twesswde37tpfw2z4v5a8jmsnwdwsg8sc2hdq302ud7j57d7dvmmt6mtc6c3fk7zkdyx5mrhq8fvyw8xp0dlgzl8kkwawt9j7m9z5adwpuskfscg5fyrpc0sca8lydtxfw7hfaaydtg88ayqj6y89a9jllhupn3hzgp7vq6d


## Test vectors for version 2

Provisional test vectors for key source $\mathtt{0x02}$ of the version 2
payload are published with the version 2 text of zcash/zips#1413
[^zips-1413-v2]. The two strings below add key sources $\mathtt{0x00}$ and
$\mathtt{0x02}$ with the key material of the version 1 vectors. Byte strings
are in hex.

- A is the member quoted above: its $\mathsf{sk}$, $\mathsf{ak}$ and signing
  share, with the birthday height 2700000.
- B holds the raw Orchard full viewing key ($\mathsf{ak}$, $\mathsf{nk}$,
  $\mathsf{rivk}$) of the 3-of-5 ZIP 2005 group of the version 1 vectors, and
  the signing share of that group's member 1 under the identifier 256. It is
  a vector of the encoding only: the share is not the share of the identifier
  256, which no step of [Restore, version 2] notices when the wallet knows no
  verifying share of the group.

Test vectors for the envelope, for a remaining part in each mode, and for
registered and private-use items, will follow with an implementation. Test
vectors for the backup file are in [Test vectors for the backup file].

A. Version 2, key source $\mathtt{0x00}$, short identifier 1, 2-of-3, Mainnet, birthday
height 2700000, as a string:

    payload 020000020300012caaa5adfcad8ac0bb8c50dcdbbc1e954825be403f97f44bc1d6c543e15d4f3be032290024cf13c724758886048e05834d5d0a644e9c8e155d7e0bac3f25586b6caa11c3a0dffa5f21db1b2c6cc72d9b6af9d777884c2c7d3e06ad479732bec46cee110d
    string  zkfrost1qgqqqqsrqqqje2494h72mzkqhwx9phxmhs0f2jp9heqrl9l5f0qad32ru9w57wlqxg5sqfx0z0rjgavgsczgupvrf4ws5ezwnj8p2ht7pwkr7f2cddk25ywr5r0l5hepmvdjcmx89kdk47whw7yyctra8cr263uhx2lvgm8wzyxsk7mugr

B. Version 2, key source $\mathtt{0x02}$, full identifier 256, 3-of-5, Testnet, birthday
height 0, as a string:

    payload 0202010305010001000000000000000000000000000000000000000000000000000000000000564f68c2939b5efe65c64d7fe0322e7aaebd7377cace2dbaf5dd3994fc8adc2400000000ad41d8ea8eaf287da6792d386d19e5238f5270ae710802c012f1bbac99e113109e4098939d8c69ce2e246d921bb3d7252a42f9aed39c81245c10606e92eb2922e22aa26359b531bd3e0118a865c7f33d2dda5fd7e30d8497d3bb7bf10fb46325
    string  zkfrost1qgpqzqc9qyqqzqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqq4j0drpf8x67lejuvntluqezu74wh4eh0jkw9ka0thfejn7g4hpyqqqqqq9dg8vw4r409p76v7fd8pk3nefr3af8ptn3pqpvqyh3hwkfncgnzz0ypxynnkxxnn3wy3keyxan6ujj5she4mfeeqfytsgxqm5jav5j9c325f34ndf3h5lqzx9gvhrlx0fdmf0a0ccdsjta8wmm7y8mgce9kskuxa

## Test vectors for the backup file

The values below were computed with an implementation of this text, and
checked against a second computation written from ZIP 32 and ZIP 325 and
against `age` 1.3.2. Identities are in the text form of age: Bech32 in uppercase with the human-readable part
`AGE-SECRET-KEY-PQ-` [^age-mlkem768x25519].

The wallet seed is the 64 bytes $\mathtt{0x00}, \mathtt{0x01}, \ldots,
\mathtt{0x3F}$. Its encryption keys (see [Encryption key from a wallet seed])
are:

    Mainnet  AGE-SECRET-KEY-PQ-16G5NCNQRD9UUPY9GP6XZLXQ8PMUKNAGJV74KCQ2ZX840Z0GNJP7S0SW5ES
    Testnet  AGE-SECRET-KEY-PQ-1YDSR3RXAVKPTHWFTHJQHKWLD2Q5YLYFMSVZAD2XTPUD982WCRZWQR6CRZM

With the prefix `x`, the name of a backup file opened by either key (see
[File name of a backup file]) is `x-seed-natt0kp8.zbackup`.

The backup mnemonic whose 16 bytes are zero is
`zbackup abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about`.
Its encryption key (see [Encryption key from a backup mnemonic]) is:

    AGE-SECRET-KEY-PQ-1SHSSJLC0H5ALRZ5X37TNKM3V657LEYN5H2Q5PGA8Z06HE5VFVULSLYN8SQ

Test vectors for whole backup files, written by an implementation of this
text and by `age`, will follow.

# Open issues

- **Passphrase.** Encrypting a backup file under a passphrase, with a
  memory-hard KDF, as Zafe does for its backups. It would be another stanza
  type of the backup file, such as the scrypt stanza of age
  [^age-scrypt], which age requires to be the only stanza of its file. It
  needs a choice of parameters.
- **Subject of the key derived from a wallet seed.** The subject
  $\texttt{“ZodlFROSTMemberBackupV1”}$ carries a wallet prefix and a version
  number, as ZIP 325 advises for a private-use subject
  [^zip-0325-private-use]. Should this key be a standardised metadata
  protocol of ZIP 325 instead of a private-use key?
- **The version 1 reject vector named "version 2".** One string of
  `frost_member_backup_decoding` is a version 1 payload with the first byte
  $\mathtt{0x02}$, expected to be rejected as an unsupported version (see
  [Test vectors]). With version 2 defined, it is rejected for another reason,
  its network and identifier form. It has to be replaced by a string whose format version
  no version of this ZIP uses, such as $\mathtt{0x03}$, and the count of
  rejected strings in [Test vectors] checked again.
- **Integrity of an unencrypted envelope.** An envelope under wrapper
  $\mathtt{0x00}$ has no checksum, and neither has its text form, nor a
  remaining part in the clear. Should they carry one?
- **Items that a decoder has to understand.** Every item can be skipped. ZIP
  316 reserves a range of typecodes that a consumer has to understand or else
  reject the whole key [^zip-0316-metadata]. This ZIP has no such range: a
  change that an older wallet must not ignore takes a new format version. Is
  a range needed?
- **Human-readable part.** `zkfrost` is kept for compatibility with existing
  backups. Should a later version use a more neutral one?

# References

[^BCP14]: [Information on BCP 14 — "RFC 2119: Key words for use in RFCs to Indicate Requirement Levels" and "RFC 8174: Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words"](https://www.rfc-editor.org/info/bcp14)

[^protocol-networks]: [Zcash Protocol Specification, Version 2026.8.0 [NU6.3]. Section 3.12: Mainnet and Testnet](protocol/protocol.pdf#networks)

[^protocol-updated-orchardkeycomponents]: [Zcash Protocol Specification, Version 2026.8.0 [NU6.3]. Section 4.2.3: Orchard Key Components](protocol/protocol.pdf#orchardkeycomponents)

[^protocol-concretespendauthsig]: [Zcash Protocol Specification, Version 2026.8.0 [NU6.3]. Section 5.4.7.1: Spend Authorization Signature (Sapling and Orchard)](protocol/protocol.pdf#concretespendauthsig)

[^protocol-pallasandvesta]: [Zcash Protocol Specification, Version 2026.8.0 [NU6.3]. Section 5.4.9.6: Pallas and Vesta](protocol/protocol.pdf#pallasandvesta)

[^protocol-orchardfullviewingkeyencoding]: [Zcash Protocol Specification, Version 2026.8.0 [NU6.3]. Section 5.6.4.4: Orchard Raw Full Viewing Keys](protocol/protocol.pdf#orchardfullviewingkeyencoding)

[^protocol-concreteblake2]: [Zcash Protocol Specification, Version 2026.8.0 [NU6.3]. Section 5.4.1.2: BLAKE2 Hash Functions](protocol/protocol.pdf#concreteblake2)

[^protocol-spendauthsig]: [Zcash Protocol Specification, Version 2026.8.0 [NU6.3]. Section 4.15: Spend Authorization Signature (Sapling and Orchard)](protocol/protocol.pdf#spendauthsig)

[^zip-0173]: [ZIP 173: Bech32 Format](zip-0173.rst)

[^zips-1413-v2]: [zcash/zips#1413: the version 2 payload as revised on 2026-11-20 and 2026-11-21](https://github.com/zcash/zips/issues/1413#issuecomment-7068742989)

[^zips-1413-v2-restore]: [zcash/zips#1413: the version 2 payload, decoding and restore of 2026-11-20](https://github.com/zcash/zips/issues/1413#issuecomment-7046243157)

[^zip-0312]: [ZIP 312: FROST for Spend Authorization Multisignatures](zip-0312.rst)

[^zip-0312-key-generation]: [ZIP 312: FROST for Spend Authorization Multisignatures — Key Generation](zip-0312.rst#key-generation)

[^frost-vss]: [RFC 9591: The Flexible Round-Optimized Schnorr Threshold (FROST) Protocol for Two-Round Schnorr Signatures. Appendix C.2: Verifiable Secret Sharing](https://www.rfc-editor.org/rfc/rfc9591.html#name-verifiable-secret-sharing)

[^cocktail-dkg-round2]: [COCKTAIL-DKG: Distributed Key Generation protocol for FROST. Round 2: Share Decryption and Verification](https://c2sp.org/cocktail-dkg)

[^zip-0316]: [ZIP 316: Unified Addresses and Unified Viewing Keys](zip-0316.rst)

[^zip-0316-experimental]: [ZIP 316: Unified Addresses and Unified Viewing Keys — Experimental Usage](zip-0316.rst#experimental-usage)

[^zip-0316-metadata]: [ZIP 316: Unified Addresses and Unified Viewing Keys — Metadata Items](zip-0316.rst#metadata-items)

[^zip-2005]: [ZIP 2005: Ironwood Quantum Recoverability](zip-2005.md)

[^zip-2005-usagewithfrost]: [ZIP 2005: Ironwood Quantum Recoverability — Usage with FROST](zip-2005.md#usagewithfrost)

[^zip-2005-threat-model]: [ZIP 2005: Ironwood Quantum Recoverability — Threat model](zip-2005.md#threatmodel)

[^zip-2005-dlog-before]: [ZIP 2005: Ironwood Quantum Recoverability — Effects of discrete-logarithm-breaking attacks before the switch to the Recovery Protocol](zip-2005.md#effectsofdiscrete-logarithm-breakingattacksbeforetheswitchtotherecoveryprotocol)

[^Bitcoin-CompactSize]: [Variable length integer. Bitcoin Wiki](https://en.bitcoin.it/wiki/Protocol_documentation#Variable_length_integer)

[^rfc4648-base64]: [RFC 4648: The Base16, Base32, and Base64 Data Encodings. Section 4: Base 64 Encoding](https://www.rfc-editor.org/rfc/rfc4648.html#section-4)

[^rfc4648-base64url]: [RFC 4648: The Base16, Base32, and Base64 Data Encodings. Section 5: Base 64 Encoding with URL and Filename Safe Alphabet](https://www.rfc-editor.org/rfc/rfc4648.html#section-5)

[^xchacha]: [XChaCha: eXtended-nonce ChaCha and AEAD_XChaCha20_Poly1305. Internet-Draft draft-irtf-cfrg-xchacha-03](https://datatracker.ietf.org/doc/html/draft-irtf-cfrg-xchacha-03)

[^age]: [The age file encryption format, `age-encryption.org/v1`. C2SP](https://github.com/C2SP/C2SP/blob/main/age.md)

[^age-mlkem768x25519]: [The age file encryption format — The MLKEM768-X25519 (i.e. X-Wing) hybrid post-quantum recipient type](https://github.com/C2SP/C2SP/blob/main/age.md#the-mlkem768-x25519-ie-x-wing-hybrid-post-quantum-recipient-type)

[^age-scrypt]: [The age file encryption format — The scrypt recipient type](https://github.com/C2SP/C2SP/blob/main/age.md#the-scrypt-recipient-type)

[^age-armor]: [The age file encryption format — ASCII armor](https://github.com/C2SP/C2SP/blob/main/age.md#ascii-armor)

[^rfc9180]: [RFC 9180: Hybrid Public Key Encryption](https://www.rfc-editor.org/rfc/rfc9180.html)

[^hpke-pq]: [Post-Quantum and Post-Quantum/Traditional Hybrid Algorithms for HPKE. Internet-Draft draft-ietf-hpke-pq](https://datatracker.ietf.org/doc/draft-ietf-hpke-pq/)

[^fips203]: [FIPS 203: Module-Lattice-Based Key-Encapsulation Mechanism Standard](https://doi.org/10.6028/NIST.FIPS.203)

[^rfc7748]: [RFC 7748: Elliptic Curves for Security](https://www.rfc-editor.org/rfc/rfc7748.html)

[^zip-0032-wallet-seeds]: [ZIP 32: Shielded Hierarchical Deterministic Wallets — Specification: Wallet seeds](zip-0032.rst#specification-wallet-seeds)

[^zip-0032-registered-key-derivation]: [ZIP 32: Shielded Hierarchical Deterministic Wallets — Specification: Registered key derivation](zip-0032.rst#specification-registered-key-derivation)

[^zip-0032-seed-fingerprints]: [ZIP 32: Shielded Hierarchical Deterministic Wallets — Seed Fingerprints](zip-0032.rst#seed-fingerprints)

[^zip-0325-metadata-key-tree]: [ZIP 325: Account Metadata Keys — Metadata key tree](zip-0325.md#metadatakeytree)

[^zip-0325-private-use]: [ZIP 325: Account Metadata Keys — Private-use metadata keys](zip-0325.md#private-usemetadatakeys)

[^bip-0039]: [BIP 39: Mnemonic code for generating deterministic keys](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki)

[^bip-0350]: [BIP 350: Bech32m format for v1+ witness addresses](https://github.com/bitcoin/bips/blob/master/bip-0350.mediawiki)

[^zkool-forum]: [Hanh, Zcash Community Forum, 2026-07-15](https://forum.zcashcommunity.com/t/56660/5)

[^zkool-backup]: [Zkool: the zkfrost member backup string](https://github.com/hhanh00/zkool2/blob/4ad5a89e09e939a415dd7376ec94f0a241f98c8c/rust/src/frost/backup.rs)

[^zkool-padding-fix]: [hhanh00/zkool2#1468: Bech32 decoders reject non-canonical padding](https://github.com/hhanh00/zkool2/pull/1468)

[^zcash-test-vectors-pr]: [zcash/zcash-test-vectors#136: Add test vectors for FROST member backup strings](https://github.com/zcash/zcash-test-vectors/pull/136)

[^frost-refresh]: [ZcashFoundation/frost: frost-core `keys::refresh`](https://github.com/ZcashFoundation/frost/blob/main/frost-core/src/keys/refresh.rs)

[^zip-0316-birthday]: [ZIP 316: Unified Addresses and Unified Viewing Keys — Viewing Key Birthday Height Metadata](zip-0316.rst#viewing-key-birthday-height-metadata)

[^frost-zec-backup]: [frost-zec: member backup strings](https://github.com/OpenPGP-Wallet/frost-zec/blob/master/src/backup.rs)

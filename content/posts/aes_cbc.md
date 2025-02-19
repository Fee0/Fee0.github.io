+++
title = "AES-CBC: How to not shoot yourself in the foot"
date = "2025-02-19"

[extra]
repo_view = false
comment = false
+++

# Intro
Before the advent of [AEAD](https://de.wikipedia.org/wiki/Authenticated_Encryption) modes, there were usually distinct cryptographic primitives for encryption and authentication. When designing protocols engineers were to their own which primitives to use and which to combine, or not to combine. This led to countless errors on the way that totally break security of many services. One often used primitive for encryption is [AES](https://de.wikipedia.org/wiki/Advanced_Encryption_Standard) in [CBC](https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation#CBC) mode. Unfortunately, this primitive needs some careful considerations and there are multiple way to shoot yourself in the foot.

Even after reading this post the same old rule applies: 
> DO NOT IMPLEMENT YOUR OWN CRYPTO! ... or do it only to learn and understand.

# Authentication
AES-CBC can provide confidentiality to a message. That’s it. Nothing more. It does not guarantee that a message will arrive like you send it. Maybe some man-in-the-middle (MITM) intercepted the message and cut off half of it. Depending on the messages' content this can be a major flaw.

Even worse, no authentication can make this block cipher vulnerable to [padding oracle](https://robertheaton.com/2013/07/29/padding-oracle-attack/) attacks. A padding oracle attack can be performed if the message needs padding which is usually the case with block ciphers arbitrary length message needs to be supported but the block cipher only works on multiples of its block size. When receiving an encrypted message, the padding needs to be verified by the receiver. Depending on if the padding is valid or invalid the server might response with an error message or a normal response. This error message can now be used to as an Oracle by the attack to provide information about if a message is considered valid. By changing the last block of the message, the adversary can decipher the encrypted message byte-by-byte by using the oracle to validate the modification to the cipher text.

When using some kind of authentication, over the ciphertext, the integrity can be verified before decryption and a modified ciphertext wouldn't even get to the decryption stage. This is the so called [doom principle](https://moxie.org/2011/12/13/the-cryptographic-doom-principle.html): 

> *"If you have to perform any cryptographic operation before verifying the MAC on a message you’ve received, it will somehow inevitably lead to doom."*

# Tag verification
If authentication is used one common mistake is to introduce timing attacks vulnerabilities when verification the messages' Tag. Exiting the verification process early when a wrong byte is encountered is a common way to introduce such a vulnerability. For example, comparing tags byte-by-byte and exiting on a wrong byte will make the runtime of the function shorter compared to a successful comparison. By measuring the time until a receiver response to a message the valid Tag can be guessed byte-by-byte. Here is an example for a vulnerable C-like verification function.

```rust
fn verify_tag(bad_tag: &[u8], correct_tag: &[u8]) -> bool {
    for i in 0..bad_tag.len() {
        if bad_tag[i] != correct_tag[i] {
            // Early exit leaks timing information!
            return false;
        }
    }
    true
}
```

However, in modern languages like Rust, it might not be that obvious what is actually happening because operators are overloaded:

```rust
fn verify_tag(bad_tag: &[u8], correct_tag: &[u8]) -> bool {
    // NOT constant-time!
    bad_tag == correct_tag
}
```

Usually, there are libraries which provide constant time functionality:

```rust
use subtle::ConstantTimeEq;

fn verify_tag(bad_tag: &[u8], correct_tag: &[u8]) -> bool {
    bad_tag.ct_eq(correct_tag).into()
}
```

# IV

CBC mode uses an [IV](https://en.wikipedia.org/wiki/Initialization_vector) (Initialization Vector). The IV not secret and can be transmitted alongside the message in plain text. However, then it must be included authentication. Otherwise, an adversary can freely manipulate the first plaintext block through manipulating the IV. 

The IV is used to make every message appear different, even when it’s the same plain text and key. Therefore, the IV must be completely indistinguishable from randomness for every message under the same key. Simple operations like incrementing the IV are not enough.

# End
It is possible to implement AES-CBC in a secure way, but there are pitfalls even when stiching together secure cryptographic primitives. Modern AEAD protocols (e.g. [AES-GCM-SIV](https://en.wikipedia.org/wiki/AES-GCM-SIV)) specify battle tested ways of combining encryption and authentication primitives and prevent many footguns by design.


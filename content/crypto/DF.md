+++
title = "Diffie-Hellman"
weight = 5
+++

The use of Diffie-Hellman allows you to exchange keys in a public environment (on an open network) to subsequently communicate securely. The two parties involved in this exchange will each create a secret in parallel, which will be randomly generated. This secret will undergo calculations to generate the associated public key. This key, held by both parties, will be exchanged with each other. Once in possession of this key, each will again perform a calculation by adding their secret. These different steps ultimately allow both parties to have exactly the same key, but their respective secret never appears in the public exchanges, making it impossible for someone intercepting their communication to find the value of this secret key. The goal of Diffie-Hellman is thus to create the same key to communicate later without having to exchange private information in a public space. Here is a diagram to visualize these exchanges:

![df1.png](../df1.png)

## Implementation of Diffie-Hellman
In this section, we recommend creating a new project via `cargo new`. We will use the following crates, to be added to your Cargo.toml:


- `rand_core` = `{ version = "0.6.4", features = ["getrandom"] }`
- `x25519-dalek` = `2.0.0`


`rand_core` will be used for secret generation and will rely on a feature of the crate `getrandom`. This crate is dedicated to generating random numbers based on the operating system on which it is called. `x25519-dalek` is the crate that implements Diffie-Hellman.

From what we have seen previously, we will need to generate a secret/public key pair for each party. Subsequently, the public key of each will be sent to the other to create the shared key. Here is the corresponding code:
```rust
use x25519_dalek::{EphemeralSecret, PublicKey};
use rand_core::OsRng;

fn main() {
    // Create the first pair of secret and public key with an ephemeral secret (can be used just one time)
    let first_secret = EphemeralSecret::random_from_rng(OsRng);
    let first_secret_public_key = PublicKey::from(&first_secret);

    // Create the second pair of secret and public key
    let second_secret = EphemeralSecret::random_from_rng(OsRng);
    let second_secret_public_key = PublicKey::from(&second_secret);

    // Create the two same shared keys
    let first_shared = first_secret.diffie_hellman(&second_secret_public_key);
    let second_shared = second_secret.diffie_hellman(&first_secret_public_key);

    // Confirm that the two generated shared keys are the same
    assert_eq!(first_shared.as_bytes(), second_shared.as_bytes());
}
```

The `EphemeralSecret` structure calls the "random_from_rng" method, which takes a random number provided by `OsRng` as a parameter. This first secret is then passed as a parameter to the `PublicKey` structure within the "from" method to give us a public key. This key is then assumed to be transferred between the two parties. Each will subsequently use the other's public key as a parameter of the `diffie_hellman` method applied to their respective secret. Two shared keys result from this process. After running the program, no errors are detected, indicating that our final instruction: `assert_eq!` is valid and that the two keys are identical.

![df2.png](../df2.png)

The two shared keys held by our two parties are now usable to encrypt their future communications. The corresponding code is available in the next section.

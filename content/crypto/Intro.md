+++
title = "Introduction"
weight = 1
+++
In this section, we'll look at the practical aspects of cryptography, focusing on key elements such as symmetric, asymmetric and Diffie-Hellman encryption, hash functions and ending with SSL/TLS sockets. We won't go into detail about how these elements work, as cryptography itself would take up an entire course. Nevertheless, there will be a theoretical section followed by practical implementations.

Implementing cryptographic functions or hashes within our malware will be useful, in particular to avoid transmitting our data streams in clear text in the case of reverse shell or when developing the practical case on ransomware.

We have selected RustCrypto as the source of crates for this course, as this project is used and maintained by a large number of users. What's more, its updates are audited by external parties to ensure that the implementations are free of security flaws. The openssl crate will be used for sockets.

## Using RustCrypto
The GitHub directory available here : RustCrypto contains a set of crates relating to cryptography. In particular, we'll be using :
 
- RSA for asymmetric encryption
- SHA-2 for the hash function
- AES-GCM for symmetric encryption
- x25519_dalek for Diffie-Hellman

To import the crates into your projects, simply select the one you want from those available on the RustCrypto GitHub, then access its crates.io link:

![intro1.png](../intro1.png)

On this platform, you'll find information about the crate, such as the last update, file size, implementation examples, dependencies and available versions. In this example, we'll be using RSA. To import it into your project, simply add the following line to our Cargo.toml dependency file, which is present once the project has been created via cargo new :

![intro2.png](../intro2.png)

The procedure will be the same for the other crates discussed in this chapter, except for x25519_dalek, which will be available via another GitHub directory: 25519-dalek. We advise you to read the documentation for the crates we'll be using, as you'll find the answers to your questions much more easily than by searching the Internet. The answers will also be of higher quality, enabling you to really understand how the functions, methods, traits or data used work.
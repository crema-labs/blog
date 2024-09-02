---
date: 2024-09-02
title: SXG Verification
---
# Proposal for Efficient zk Proofs in SXG Verification

A design proposal for improving performance of sxg zk proofs verification.

## Contents


- [Proposal for Efficient zk Proofs in SXG Verification](#proposal-for-efficient-zk-proofs-in-sxg-verification)
  - [Contents](#contents)
  - [Abstract](#abstract)
  - [V0 : Direct Approach](#v0--direct-approach)
    - [Pros](#pros)
    - [Cons](#cons)
  - [V1 : Using Merkle Proofs to verify the payload](#v1--using-merkle-proofs-to-verify-the-payload)
  - [V2 : Multi Level Proof Generation](#v2--multi-level-proof-generation)
    - [Pros](#pros-1)
      - [Constraints](#constraints)
      - [Improvement from V0 to V2](#improvement-from-v0-to-v2)
    - [Cons](#cons-1)
  - [Conclusion](#conclusion)


## Abstract

The demo for the zk proofs of sxg signature intends to build an price oracle on chain with data from a trusted off-chain source. The example does not necessarily needs privacy or data obscurity. The objective of the example chosen is to create a protocol that utilizes the zk proofs to verify that a certain piece of content is signed by a certain key from the trusted source to prove authenticity of the content. This example also pushes the design to be fairly fast to be able to generate and verify the proofs in a reasonable time to accommodate frequent changes in prices of the selected asset.

- The objective of the proposal is to reduce the amount of the time taken to generate proofs for a verifiable sxg signature.
- The following sections describe the version wise decisions to justify the proposed approach.

## V0 : Direct Approach

The following steps are taken to generate the proof for the sxg signature:

1. Client sends the request to the server for data.
2. Server generates necessary response and sends it to the client with the sxg signature.
> ***Example***
> ```
> format version: 1b3
> request:
>   method: GET
>   uri: https://signed-exchange-testing.dev/sxgs/valid.html
>   headers:
> response:
>   status: 200
>   headers:
>     Digest: mi-sha256-03=+g9t4KB79Gt4iWA+SHo8bz9HIOgkR3PA2Nv9uLOr9JA=
>     Content-Type: text/html; charset=utf-8
>     Content-Encoding: mi-sha256-03
> signature: label;cert-sha256=*ZLprKDvE6RF97s+TApgIHtcTMjt6SKIJPQz5t1vS7ok=*;cert-url="https://signed-exchange-testing.dev/certs/cert.cbor";date=1723831201;expires=1723917601;integrity="digest/mi-sha256-03";sig=*MEUCIDxjv3GBR/xTB2SGIQIVmaxcWKciWEH+2C1sPCyL7BhLAiEAgkzB4HRDu1/elOc4hvAa4mdTfoFRRihmGJe5TCd0dow=*;validity-url="https://signed-exchange-testing.dev/validity.msg"
> header integrity: sha256-A/3QHzsuh6GDLO/nFxG3YtiawxuroAsQdf4bwFOO6eU=
> payload [264 bytes]:
> <!DOCTYPE html>
> <html>
>   <head>
>     <title>An SXG Content</title>
>     <meta charset="utf-8">
>     <meta name=supported-media content="only screen and (max-width: 640px)">
>   </head>
>   <body>
>     <h1>
>       This is an SXG content example.
>     </h1>
>   </body>
> </html>
> ```
3. Get required information for serialization of msg.
    - Client will parse the sxg signature and extract the necessary information.
    - Client will fetch the certificate from the `cert-url` and parse it.

4. Proof generation
    inputs: (all inputs will be are expected to be in the form of bytes)
    - sxg signature , `sxg`
    - public key, `pk`
    - payload, `p`
    - [`cert-sha256`,`value-url`,`Date`,`Expires`,`Digest`,`Integrity`,`Content-Type`,`Content-Encoding`]
    - data to be verified or stored on chain, `d`

   Circuit Flow:
   - Step 1
       - Verify data inclusion of `d` in `p` 
   - Step 2
       - Verify `Digest` by performing mi-sha256-03 on `p`
   - Step 3
       - Construct `msg` by serializing the data as mentioned in the [spec](https://wicg.github.io/webpackage/draft-yasskin-http-origin-signed-responses.html#name-signature-validity)
   - Step 4
       - Verify `sxg` signature by performing ECDSA verification on `sxg` using `pk` and `msg` on p256 curve.

5. Proof verification will out output validity of the signature and outputs the data to be stored on chain `d`.

### Pros
---
1. Easy and plug and play approach for enabling SXG's on a domain service.
2. The approach is simple and easy to implement the server.

### Cons
---
1. The above approach is not efficient and hardly scalable as it requires to perform an expensive operation `sha256` on select chunks of payload multiple times on the payload.
2. Very high number of constraints

    | **ALgorithm** | **Frequency** | **Constraints Per Usage** |
    |:-----------------:|:---------------:|:--------:|
    |         p256          |   1              |   1,972,905       |
    |            sha256       |       (len(`p`)/chunkSize ) + 1         |          540/b |
    |           inclusion        |     1           |   len(`d`)     |
   
Rough estimate of the constraints for the above approach is 
- 1,972,905 + 540 * chunkSize * 1024 * ((len(`p`)/chunkSize ) + 1) + len(`d`)

> **Example :**
>
>   Payload : 2mb
>
>   Chunk Size : 16kb
>
>   Data to be verified : 8 bytes
> 
>     => 1,972,905 + 540/b * 16 * 1024 * (2000kb/16kb) + 8
>     => 1,107,892,913


>Thus making the approach impractical for large payloads.



## V1 : Using Merkle Proofs to verify the payload

This method did not add any significant benefit to performance as mi-sha256-03 uses a right skewed merkle tree to generate the digest. The proof generation and verification is still  as expensive as V0 for the large payloads while verifying last few bytes of the payload. 
- The construction of the Merkle tree prioritizes strengthening data integrity over optimizing for inclusion efficiency.

From mice-03 [spec](https://datatracker.ietf.org/doc/html/draft-thomson-http-mice-03)
```
       proof(A)
         /\
        /  \
       /    \
      A    proof(B)
            /\
           /  \
          /    \
         B    proof(C)
                /\
               /  \
              /    \
             C    proof(D)
                    |
                    |
                    D

           Figure 1: Proof structure for a message with 4 blocks
````


## V2 : Multi Level Proof Generation

THe following is the key part of the proposal to reduce the constraints and improve the performance of the proof generation and verification.

1. The client requests the data from the server.
2. The server generates the response , hashes it using mimc-sponge and sends the response to the client with the content containing only the hash of original payload.

> ***Example***
> ```
> format version: 1b3
> request:
>   method: GET
>   uri: https://signed-exchange-testing.dev/sxgs/valid.html
>   headers:
> response:
>   status: 200
>   headers:
>     Digest: mi-sha256-03=+bz9HIOgkR3PA2Nv9uLOr9JAg9t4KB79Gt4iWA+SHo8=
>     Content-Type: text/html; charset=utf-8
>     Content-Encoding: mi-sha256-03
> signature: label;cert-sha256=*ZLprKDvE6RF97s+TApgIHtcTMjt6SKIJPQz5t1vS7ok=*;cert-url="https://signed-exchange-testing.dev/certs/cert.cbor";date=1723831201;expires=1723917601;integrity="digest/mi-sha256-03";sig=*MEUCIDxjv3GBR/xTB2SGIQIVmaxcWKciWEH+2C1sPCyL7BhLAiEAgkzB4HRDu1/elOc4hvAa4mdTfoFRRihmGJe5TCd0dow=*;validity-url="https://signed-exchange-testing.dev/validity.msg"
> header integrity: sha256-A/3QHzsuh6GDLO/nFxG3YtiawxuroAsQdf4bwFOO6eU=
> payload [57 bytes]:
> poseidon2-03=+g9t4KB79Gt4iWA+SHo8bz9HIOgkR3PA2Nv9uLOr9JA=
> ```
3. Server caches the original payload with key as the hash of the payload until expiry of the signature.
4. Client fetches the original payload via the hash.
> ***Example***
> ```
> format version: 1b3
> request:
>   method: GET
>   uri: https://signed-exchange-testing.dev/sxgs/poseidon2-03=+g9t4KB79Gt4iWA+SHo8bz9HIOgkR3PA2Nv9uLOr9JA=
>   headers:
> response:
>   status: 200
>   headers:
>     Content-Type: text/html; charset=utf-8
> payload [264 bytes]:
> <!DOCTYPE html>
> <html>
>   <head>
>     <title>An SXG Content</title>
>     <meta charset="utf-8">
>     <meta name=supported-media content="only screen and (max-width: 640px)">
>   </head>
>   <body>
>     <h1>
>       This is an SXG content example.
>     </h1>
>   </body>
> </html>
> ```
5. Proof generation
    inputs: (all inputs will be are expected to be in the form of bytes)
    - sxg signature , `sxg`
    - public key, `pk`
    - original payload, `p`
    - hashed payload, `hp`
    - [`cert-sha256`,`value-url`,`Date`,`Expires`,`Digest`,`Integrity`,`Content-Type`,`Content-Encoding`]
    - data to be verified or stored on chain, `d`
  
   Circuit Flow:    
    - Step 1
        - Verify data inclusion of `d` in `p` 
    - Step 2
        - Verify `Digest` by performing mi-sha256-03 on `hp` which requires only 1 sha256 operation. 
    - Step 3
        - Verify poseidon2 hash of `p` is equal to `hp`.
    - Step 4
        - Construct `msg` by serializing the data as mentioned in the [spec](https://wicg.github.io/webpackage/draft-yasskin-http-origin-signed-responses.html#name-signature-validity) for `hp`
    - Step 4
        - Verify `sxg` signature by performing ECDSA verification on `sxg` using `pk` and `msg` on p256 curve.

### Pros
---
- Reduces constriants by a significant magnitude thus reducing time for proof generation.
#### Constraints

  | **Algorithm** | **Frequency** | **Constraints Per Usage** |
  |:-----------------:|:---------------:|:--------:|
  |         p256          |   1              |   1,972,905       |
  |            sha256       |       1         |          540/b|
  |           poseidon2        |     1           |   10/b     |
  |           inclusion        |     1           |   len(`d`)    |  

Rough estimate of the constraints for the above approach is 
- 1,972,905 + 540 * 32 + 10 * len(`p`) + len(`d`)
  
> **Example :**
>
>   Payload : 2mb
>
>   Chunk Size : 16kb
>
>   Data to be verified : 8 bytes
>  
>       => 1,972,905 + 540 * 32 + 10 * 2000 * 1024 + 8
>       => 22,470,193

#### Improvement from V0 to V2
- constraints for V0 : 1,107,892,913
- constraints for V2 : 22,470,193


>The constraints have been reduced by 98% from V0 to V2, thus making the approach practical for large payloads.


### Cons
---
1. The approach requires the server to cache the original payload.
2. Requires the client to make an additional request to fetch the original payload.
3. The approach is not plug and play as V0.


## Conclusion

V2 offers a more scalable and efficient method for proving SXG signatures using zk proofs. By leveraging poseidon instead of the typical sha256 operations and utilizing a two-step proof generation, this approach significantly reduces the constraint count and facilitates faster proof generation. This makes it an ideal solution for high-frequency applications such as on-chain price oracles. However, additional considerations regarding caching and potential edge cases should be addressed during implementation to ensure robustness.

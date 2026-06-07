# The Uselessness that Changed the World

In 1940, G.H. Hardy wrote in *A Mathematician's Apology*:

> "I have never done anything 'useful.' No discovery of mine has made, or is likely to make, directly or indirectly, for good or ill, the least difference to the amenity of the world."

Hardy was a number theorist. He studied prime numbers, divisibility, and modular arithmetic — the purest of pure mathematics. He was proud of its uselessness.

Thirty-seven years later, Ron Rivest, Adi Shamir, and Leonard Adleman published "A Method for Obtaining Digital Signatures and Public-Key Cryptosystems." The RSA algorithm rests entirely on number theory: modular exponentiation, Fermat's little theorem, the hardness of factoring large integers. The "useless" discipline became the foundation of all secure internet communication.

Today, number theory powers:
- **TLS/SSL**: Every secure web connection uses modular arithmetic (RSA, ECDH, or both).
- **Cryptocurrency**: Bitcoin's digital signatures (ECDSA) use elliptic curves over finite fields.
- **End-to-end encryption**: Signal, WhatsApp, and iMessage use number-theoretic primitives.
- **Code signing**: Operating system updates are verified with public-key cryptography.
- **Blockchain**: Proof-of-work relies on cryptographic hash functions.

This chapter covers the number theory you need to implement these algorithms efficiently. We start with modular arithmetic and exponentiation, build up through the extended Euclidean algorithm and Montgomery multiplication, and end with finite fields, cryptography, hashing, random number generation, and error correction.

Along the way, we measure everything. A modular inverse takes 330 ns with the naive approach, 180 ns with binary exponentiation, 170 ns with extended Euclid, and 158 ns with fully optimized Montgomery multiplication on Zen 2. These aren't academic differences — at scale, a 2× speedup in the cryptographic primitive doubles the throughput of your TLS termination.

The beautiful irony of this chapter is that the "useless" mathematics Hardy celebrated is now among the most performance-critical code on the planet. Every HTTPS request, every signed software update, every blockchain transaction runs number theory in a hot loop. Let's make it fast.

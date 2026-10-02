# Vulnerabilities in KeyPal HW Wallet

## Summary

Some AI-assisted research I have done on few HW wallet products for different vendors. It was a fun little experiment to see what AI is capable of.

### KeyPal (firmware 3.1.0) — QR decoder heap out-of-bounds write

The QR-code text-assembly routine used when decoding structured-append QR
sequences contains a heap buffer overflow. It writes up to 7 attacker-controlled
bytes past the end of a heap allocation, corrupting the adjacent chunk. The input
is delivered simply by pointing the device camera at a crafted image — no pairing,
unlock, or user confirmation required (pre-auth).

### Root cause
For a structured-append group the routine works in two steps:

1. **Size estimation** — it sums only the payload lengths of the member codes and
   allocates a buffer of that size + 1.
2. **Assembly** — it copies each member's payload into the buffer and, for every
   *missing* segment in the sequence, writes an extra one-byte separator that was
   **not** accounted for in step 1.

Each isolated gap therefore writes one uncounted byte. Once the write cursor passes
the estimated size, the remaining bounds check `estimated - written` is computed in
**unsigned** arithmetic and underflows to a huge value, so every subsequent copy is
treated as in-bounds and proceeds past the buffer. The number of overflowing bytes
equals `(#member QR codes) − 1`; the structured-append format caps this at **7 bytes**,
and their values are attacker-chosen (via the QR payloads).

By sizing the group so the vulnerable buffer is a tight 16-byte allocation, the
overflow lands directly on the next heap chunk's header. The corrupted header is
later dereferenced by the allocator: crash observed inside the heap free routine at
**PC = 0x0806E00A** (allocator functions at **0x0806E0F0** / **0x0806DFF8**).

###  PoC

I have implemented a PoC. Can share more details to vendors as the communication has been interrupted since I proved that.

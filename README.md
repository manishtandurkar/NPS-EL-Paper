# IEEE Conference Paper — Work Split

**Title:** Adaptive Secure Communication System for Unreliable and Adversarial Networks

---

## How to Build

```
pdflatex -interaction=nonstopmode paper.tex
pdflatex -interaction=nonstopmode paper.tex
```

Two passes required. Output: `paper.pdf`.

---

## Member Responsibilities

### Member 1 — Introduction, Related Work, Architecture, Conclusion

| Section | Status |
|---------|--------|
| I. Introduction | TODO — motivate problem, summarise contributions, state paper structure |
| II. Related Work | TODO — Signal DR, TLS 1.3, multipath, adaptive security, IDS literature |
| III. System Architecture | TODO — two-tier design, module map table, architecture diagram |
| X. Conclusion | TODO — summarise contributions and future work |

**Key source material (AGENTS.md):** §2 What This Project Is, §3 Problem Being Solved, §4 Key Contributions, §5 System Architecture, §19 References to Cite

---

### Member 2 — Wire Protocol, Cryptographic Design, Limitations

| Section | Status |
|---------|--------|
| IV. Wire Protocol and Connection Lifecycle | TODO — 28-byte header, message type table, sequence diagram, server re-encryption trade-off |
| V. Cryptographic Design | TODO — four-layer stack, X25519, RSA-2048, TLS 1.3, Double Ratchet (init, KDF_CK, KDF_RK equations), key lifetime table, OPENSSL_cleanse |
| IX. Limitations and Future Work | TODO — all 6 limitations from §16 of AGENTS.md |

**Key source material (AGENTS.md):** §6 Wire Protocol, §7 Connection Lifecycle, §8 Cryptographic Architecture, §9 Double Ratchet Protocol, §16 Limitations

---

### Member 3 — Adaptive Engine, IDS, Transport, Evaluation

| Section | Status |
|---------|--------|
| VI. Adaptive Engine and IDS | TODO — Metrics struct, three states + parameter table, transition thresholds + hysteresis, IDS IpRecord struct, block logic, log format, coupling |
| VII. Multipath Transport and Offline Queue | TODO — dual TCP+UDP loop, dedup ring buffer, priority queue, offline queue disk layout, plaintext rationale, security hardening |
| VIII. Evaluation | TODO — format 13-test results table, engine transition discussion, performance measurements |

**Key source material (AGENTS.md):** §10 Adaptive Engine, §11 Multipath Transport, §12 Offline Queue, §13 Priority Queue, §14 IDS, §17 Key Constants, §18 Test Results

---

## Shared Tasks

| Task | Owner | Status |
|------|-------|--------|
| Fill in author names and affiliations in `paper.tex` | All | TODO |
| Create figures (architecture diagram, state machine diagram) | Discuss | TODO |
| Final proofread and consistency check | All | TODO |

---

## References (pre-filled in paper.tex)

| Key | Citation |
|-----|----------|
| b1 | Marlinspike & Perrin — Double Ratchet Algorithm, 2016 |
| b2 | Cohn-Gordon et al. — Formal Security Analysis of Signal, IEEE EuroS&P 2017 |
| b3 | Rescorla — TLS 1.3, RFC 8446, 2018 |
| b4 | Krawczyk & Eronen — HKDF, RFC 5869, 2010 |
| b5 | Barnes et al. — MLS Protocol, RFC 9420, 2023 |
| b6 | Marlinspike & Perrin — X3DH Key Agreement, 2016 |

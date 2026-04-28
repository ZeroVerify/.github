# ZeroVerify

ZeroVerify is a privacy-preserving credential system built on zero-knowledge proofs. Users prove things about themselves (like being an enrolled student) without revealing any personal data. Credentials follow the W3C Verifiable Credentials spec; proofs use Groth16 zk-SNARKs compiled with Circom.

## How it works

```
University (Keycloak)
       │  OIDC login
       ▼
 issuer-lambda  ──── signs credential fields (BabyJubJub)
       │                 stores credential + revocation index (DynamoDB)
       ▼
    wallet  ──── stores encrypted credential in the browser
       │            generates ZK proof in-browser (snarkjs + circuits)
       │
       ▼  proof + challenge
  verifier  ──── uses verifier-go to check:
                   1. Groth16 proof validity
                   2. challenge match
                   3. expiry
                   4. revocation (W3C Bitstring Status List on S3)
                   5. BabyJubJub field signatures
```

Revocation propagation:

```
wallet  ──► revocation-lambda  ──► DynamoDB stream
                                        │
                               bitstring-updater-lambda
                                        │
                                    S3 bitstring  ◄──  verifiers
```

## Repositories

| Repo | Language | Role |
|------|----------|------|
| [wallet](https://github.com/ZeroVerify/wallet) | TypeScript / React | Browser wallet that stores encrypted credentials and generates ZK proofs in-browser via snarkjs |
| [issuer-lambda](https://github.com/ZeroVerify/issuer-lambda) | Go / AWS Lambda | Issues W3C Verifiable Credentials after OIDC login; signs each field with BabyJubJub; assigns a revocation bit index |
| [verifier-go](https://github.com/ZeroVerify/verifier-go) | Go (library) | Importable library for verifying Groth16 proofs: checks challenge, expiry, revocation, and field signatures |
| [circuits](https://github.com/ZeroVerify/circuits) | Circom | zk-SNARK circuits: `student_status` (proves enrollment without revealing identity) and `credential_revocation` (proves ownership for self-revocation) |
| [revocation-lambda](https://github.com/ZeroVerify/revocation-lambda) | Go / AWS Lambda | Accepts a revocation ZK proof from the wallet and marks the credential revoked in DynamoDB |
| [bitstring-updater-lambda](https://github.com/ZeroVerify/bitstring-updater-lambda) | Go / AWS Lambda | Triggered by DynamoDB Streams; propagates revocations to the W3C Bitstring Status List in S3 |
| [free-lambda](https://github.com/ZeroVerify/free-lambda) | Go / AWS Lambda | Reclaims expired or revoked bit indices back to the free pool so they can be reused |
| [mock-verifier](https://github.com/ZeroVerify/mock-verifier) | Go | Demo HTTP verifier that accepts proofs and streams results via SSE; useful for integration testing |
| [keycloak](https://github.com/ZeroVerify/keycloak) | Docker | Pre-configured Keycloak realm acting as the university identity provider |
| [infrastructure](https://github.com/ZeroVerify/infrastructure) | Terraform | Provisions all AWS resources (Lambda, DynamoDB, S3, API Gateway) |
| [website](https://github.com/ZeroVerify/website) | SvelteKit | Public-facing marketing and landing page |
| [scripts](https://github.com/ZeroVerify/scripts) | Shell / Node | Ceremony and deployment utilities |
| [docs](https://github.com/ZeroVerify/docs) | Typst | Technical specification, architecture diagrams, and design documents |

## Key design properties

- **No personal data leaves the browser in a proof.** The ZK circuit outputs only a challenge response, expiry timestamp, and revocation index.
- **Selective disclosure.** Field signatures let a verifier confirm specific credential fields (e.g. enrollment status) without seeing others.
- **Self-sovereign revocation.** Users revoke their own credential by generating a ZK proof of ownership; no admin action required.
- **Standard formats.** Credentials are W3C VCs; revocation uses the W3C Bitstring Status List; proofs are Groth16 produced by snarkjs.

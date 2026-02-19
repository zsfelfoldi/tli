## Trustless Log Index

This repository contains resources related to the Trustless Log Index project.

Latest specification: [EIP-7745B](https://github.com/zsfelfoldi/tli/blob/main/eip-7745b.md)

Note that this latest design is not an EIP yet (will be soon, after some polishing). This is a different approach achieving the same goals as the original [EIP-7745](https://eips.ethereum.org/EIPS/eip-7745) which is still an active EIP, current status is DFId for Glamsterdam because of its high complexity. The new design will probably replace it once proven in practice that it can achieve better performance and lower protocol complexity.

### Trustless Execution Layer REST API

Trustless Log Index is part of the Trustless Execution Layer API project, which defines endpoints for fully trustless chain access with Merkle proofs based on the assumed knowledge of a recent canonical block hash and number, which (if not available by other means) can be obtained by running a Beacon Light Client. The format of the new endpoints is similar to the Beacon REST API.

[Draft proposal](https://github.com/zsfelfoldi/tli/blob/main/execution_api_draft.md)

### ACD breakout

Bi-weekly at Tuesdays 15:00 UTC

Next session:

https://github.com/ethereum/pm/issues/1927

### Talks

EthBoulder Feb 14, 2026 [slides](https://github.com/zsfelfoldi/tli/blob/main/EthBoulder2026-slides.pdf)

EthDenver Feb 19, 2026 [slides](https://github.com/zsfelfoldi/tli/blob/main/EthDenver2026-slides.pdf)

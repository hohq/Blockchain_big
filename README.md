# EscrowTick — Escrow-Based Ticketing with Refundable No-Show Deposits

CSIT5760 Blockchain and Smart Contracts — Individual Project

| | |
|---|---|
| Student | [YOUR FULL NAME] |
| Student ID | 21316347 |
| Network | Sepolia (chain ID 11155111) |

## Summary

EscrowTick is a Solidity smart contract that lets student clubs publish ticketing events and
holds ticket payments and refundable deposits in escrow. Deposits are returned automatically
when an attendee checks in, forfeited to the organiser when an attendee does not appear, and
refunded in full if the event is cancelled. Neither the organiser nor any platform holds the
money at any point.

## Repository layout

```
big_proj/
├── PROPOSAL.md                  # Project proposal (submitted 10 Nov 2026)
├── README.md                    # This file
├── contracts/                   # Solidity sources            (to be added)
├── test/                        # Hardhat test suite          (to be added)
├── scripts/                     # Deployment and interaction  (to be added)
├── deployment/                  # deployment-information.txt  (to be added)
├── audit/                       # Slither report + manual review (to be added)
└── screenshots/                 # Deployment, verification, testing evidence (to be added)
```

## Status

- [x] Project proposal
- [ ] Contract implementation
- [ ] Test suite
- [ ] Security analysis
- [ ] Sepolia deployment
- [ ] Etherscan verification
- [ ] Final report

## Prerequisites

To be completed with the implementation (Node.js 20 LTS, Hardhat, MetaMask).

## Compile, test, deploy

To be completed with the implementation.

## Deployed addresses and verification

To be completed after deployment.

## Known limitations and security considerations

See Section 4 and Section 7 of `PROPOSAL.md`. In summary: personal data is kept off-chain;
check-in is performed by the organiser and cannot be independently verified by the contract;
the administrator can pause new purchases but can never move user funds.

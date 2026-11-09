# CSIT5760 — Blockchain and Smart Contracts

## PROJECT PROPOSAL

| Field | Value | Field | Value |
|---|---|---|---|
| Project title | EscrowTick: Escrow-Based Ticketing with Refundable No-Show Deposits for Student Societies | Student name | [YOUR FULL NAME] |
| Student ID | 21316347 | Instructor | [INSTRUCTOR NAME] |
| Submission date | 10 November 2026 | Repository | https://github.com/hohq/Blockchain_big |
| Testnet | Sepolia (chain ID 11155111) | | |

## 1. Project Title

EscrowTick: Escrow-Based Ticketing with Refundable No-Show Deposits for Student Societies

## 2. Background and Motivation

Student societies at a university run ticketing events — workshops, competitions, socials — with capacities in the tens. Because these events are informal and low-budget, organisers rely on group-chat sign-up lists and personal payment QR codes. Such lists cannot enforce capacity, so events are over-booked on paper and under-attended in practice; payments cannot be matched to names at the door; refunds must be transferred by hand. Organisers absorb the cost of empty seats while students are turned away from a full list. Neither side acts in bad faith — neither can make a credible commitment to the other. The significance lies in frequency: a deposit is a few dollars, but a society runs dozens of such events a year, and a university hosts hundreds of societies.

**Users and stakeholders.** Organisers need predictable attendance; attendees need a low-effort way to recover deposits; check-in volunteers verify attendance at the door; the student union or student affairs office approves societies and allocates funding on attendance — but is an unwilling custodian of student money.

**Limitations of existing approaches.** Group-chat lists cannot enforce capacity or handle money. Online forms cannot collect or refund money. Commercial platforms (Eventbrite, 活动行) target commercial events, charge fees, hold funds until after the event, and execute refund rules at their discretion. Payment QR codes are irreversible and cannot be linked to an attendance. University systems are closed to informal clubs. Each alternative either cannot hold money, or holds it in a way that requires trust in the holder.

**Why blockchain.** A database can record a capacity limit and a refund policy; it cannot make the *execution* of that policy independent of the party holding the money. Here the organiser holds the deposits and decides whether to return them. The student union can vouch for a society but has no appetite to hold and refund hundreds of small deposits, so the only alternatives are to trust the organiser or to invent a custodian. A contract changes the trust structure: the deposit is held by code, and the release conditions — check in and it returns, fail to appear and it is forfeited, cancel and everything is refunded — are fixed at creation and visible before payment. This is when on-chain escrow beats a database: opposed interests, no trusted intermediary, and payment released by a rule rather than a judgement.

## 3. Project Objectives

| ID | Measurable objective | Success criterion |
|---|---|---|
| O1 | **Develop** a Solidity 0.8.x contract enabling approved organisers to publish events and students to buy tickets, with capacity enforced on-chain. | An event of capacity *N* accepts exactly *N* purchases; the (*N*+1)-th reverts with a custom error; each purchase emits `TicketPurchased`. |
| O2 | **Implement** escrow settlement releasing payments only according to event outcome. | Check-in, no-show and cancellation change each party's balance by exactly the expected amount; contract balance is zero after settlement. |
| O3 | **Enforce** role-based access control and explicit lifecycle state machines. | Every state × operation pair is tested; valid transitions succeed, invalid ones revert; unauthorised accounts cannot create, check in or withdraw. |
| O4 | **Validate** the contract with a Hardhat suite covering positive, negative and boundary cases. | ≥ 50 tests pass in one run with ≥ 85% statement coverage, documented in the required Test ID / Purpose / Input / Expected / Actual / Status format. |
| O5 | **Perform** static analysis and manual security review, and remediate findings. | A Slither report with tool version is produced; every finding is triaged; all High and Medium findings are fixed and re-tested. |
| O6 | **Deploy** the contract to Sepolia and verify it on Etherscan. | Verified address, transaction hash, block number and compiler settings recorded; the README reproduces deployment and interaction. |

## 4. Proposed System and Scope

**Roles and permissions.**

| Role | Permissions |
|---|---|
| Administrator (student union) | Approve or revoke societies; pause new purchases. **Cannot move, withdraw or redirect user funds.** |
| Organiser | Create events; set capacity, price, deposit and deadlines; nominate a check-in delegate; check in attendees; finalise or cancel own events; withdraw revenue. |
| Check-in delegate | Check in attendees for one nominated event only. |
| Attendee | Buy while the sale is open; cancel before the deadline; withdraw refunds and returned deposits. |

**Principal functions.** `approveOrganizer` / `revokeOrganizer` (admin); `createEvent`, `finalizeEvent`, `cancelEvent` (organiser); `buyTicket` *(payable, requires `price + deposit`)*, `cancelRegistration`, `withdraw` (attendee); `checkIn` (organiser or delegate); `setPaused` (admin); views `getEvent`, `getTicket`, `pendingBalanceOf`.

**On-chain and off-chain.** On-chain: event parameters and state, ticket ownership and check-in status, escrowed balances, metadata URI. Off-chain: descriptions, images, venue details, attendee names and contact details — a public chain cannot delete, so identifiers would be a permanent leak, and the contract only needs to know *that* a wallet holds a ticket, not *who* the person is.

**Events and access control.** `OrganizerApproved`, `EventCreated`, `TicketPurchased`, `RegistrationCancelled`, `AttendeeCheckedIn`, `EventFinalised`, `EventCancelled`, `RefundCredited`, `Paused`/`Unpaused`. Modifiers: `onlyAdmin`, `onlyApprovedOrganizer`, `onlyEventOrganizer`, `onlyCheckInAuthority`, `whenNotPaused`.

**Ether, tokens and NFTs.** Native Ether only; no ERC-20 (a stablecoin dependency adds approval and price concerns with no benefit for small deposits). No ERC-721: a ticket must not be transferable, so a token standard would need its transfer logic overridden purely to disable it; internal records give the same guarantee with less code.

**Exclusions and assumptions.** Excluded: resale, anti-scalping caps, reputation scoring, ERC-20 or fiat rails, dispute arbitration (none is needed — whether an attendee checked in is a boolean on-chain, not a judgement), cross-institution deployment, mobile application. Assumed: the administrator approves societies as a lightweight onboarding gate; check-in is performed honestly by someone who can observe attendance; attendees hold test ETH; capacity is bounded. The design generalises to other small community groups, but evaluation is scoped to a single university.

## 5. Preliminary Smart Contract Design

| Component | Planned design | Purpose / rule |
|---|---|---|
| Structure and state | One `EventTicketing` contract; `events[eventId]`, `tickets[eventId][attendee]`, `approvedOrganizers`, `pendingBalance`, `paused` | One deployment serves every society |
| Structs and enum | `Event { organizer, capacity, sold, checkedIn, price, deposit, saleEnd, eventEnd, state, metadataURI, delegate }`; `Ticket { paid, checkedIn, refunded }`; `EventState { OnSale, SoldOut, Finished, Cancelled }` | Configuration plus explicit lifecycle |
| Access control | Modifiers `onlyAdmin`, `onlyEventOrganizer`, `onlyCheckInAuthority`, `whenNotPaused`; custom errors such as `SoldOut`, `SaleClosed`, `AlreadyCheckedIn` | Testable failure modes |
| Settlement | Purchase credits `price + deposit`; check-in credits the deposit back; finalisation credits the organiser `sold × price + (sold − checkedIn) × deposit`; cancellation credits every attendee `price + deposit` | Accounts reconcile exactly; balance reaches zero |
| Payouts and complexity | Pull payments: credits accumulate in `pendingBalance`; O(1) settlement using the `sold` and `checkedIn` counters, with no iteration over attendees | Avoids push-transfer reentrancy, gas-bounded loops and attendant denial of service |

### 5.1 Architecture or Workflow

```mermaid
stateDiagram-v2
    [*] --> OnSale: createEvent()
    OnSale --> SoldOut: capacity reached
    OnSale --> Finished: finalizeEvent() after eventEnd
    SoldOut --> Finished: finalizeEvent() after eventEnd
    OnSale --> Cancelled: cancelEvent()
    SoldOut --> Cancelled: cancelEvent()
    Finished --> [*]
    Cancelled --> [*]
```

An approved organiser creates an event, which opens for sale. Attendees pay `price + deposit`, held by the contract; before the deadline they may cancel and be credited in full. At the door the organiser or delegate checks attendees in, crediting each deposit. After the event the organiser finalises and receives revenue plus the deposits of those who did not appear. If the event is cancelled first, every attendee is credited and the organiser receives nothing. In every branch, contract liability equals the sum of pending balances, and both reach zero after all withdrawals.

## 6. Technology Stack

| Technology / version | Purpose | Rationale |
|---|---|---|
| Solidity 0.8.24 | Contract language | Checked arithmetic, custom errors |
| Hardhat 2.22.x | Compile, test, deploy | `evm_increaseTime` for deadline tests |
| OpenZeppelin Contracts 5.x — `Ownable`, `Pausable`, `ReentrancyGuard` | Access control, pause, reentrancy protection | Audited, widely used; behaviour documented |
| ethers.js 6.x / Node.js 20 LTS / MetaMask | Scripts, runtime, wallet | Integrated toolchain |
| Sepolia (11155111), Etherscan, Slither 0.10.x, solidity-coverage | Testnet, verification, analysis, coverage | Course-approved evidence for O4–O6 |

## 7. Security Considerations

| Risk | Attack / failure scenario | Mitigation | Validation |
|---|---|---|---|
| Reentrancy on withdrawal | Re-entering `withdraw()` drains other users' balances | Balance zeroed before transfer; `ReentrancyGuard` | Attacker contract: nested call reverts, balances unchanged |
| Unauthorised check-in | Anyone calls `checkIn` to reclaim a deposit without attending | Organiser or delegate only, checked per event | Negative tests from attendee, other organiser and admin |
| Overselling race | Two purchases in one block read the same `sold` | Increment and state change within one transaction | Two signers in one block: exactly one succeeds |
| Premature withdrawal | Organiser finalises before the event, taking revenue | Requires `block.timestamp > eventEnd` | Boundary tests at `eventEnd − 1`, `eventEnd`, `eventEnd + 1` |
| Refund denial of service | A loop over attendees exhausts gas, leaving some unpaid | Pull payments; no function iterates over attendees | Maximum-capacity cancellation: every refund claimable |
| Pause blocking refunds | Admin pauses and attendees cannot recover funds | Pause gates only `buyTicket` and `checkIn` | `withdraw()` succeeds while paused |

## 8. Testing Plan

| Test category | Representative scenario | Expected evidence |
|---|---|---|
| Positive | Three attendees buy, all check in, organiser finalises | Deposits returned, revenue correct, balance zero; events emitted |
| Negative | Unapproved `createEvent`; purchase after `saleEnd` | Reverts with `NotAnOrganizer` / `SaleClosed`; no state change |
| Boundary | Capacity *N* and *N*+1; purchase at and after the deadline | *N* succeeds, *N*+1 reverts; deadline behaviour defined |
| Boundary | `finalizeEvent` at `eventEnd − 1`, `eventEnd`, `eventEnd + 1` | Only the third succeeds |
| State and access | Cancellation refunds; every role × every privileged function | All credited; each unauthorised call reverts |
| Reentrancy and invariant | Nested withdrawal; arbitrary operation sequences | Nested call reverts; `balance ≥ Σ pendingBalance`, zero after settlement |

## 9. Work Plan

| Phase | Activities | Target date | Deliverable |
|---|---|---|---|
| 1. Design | State machines, accounting, access matrix | 11–13 Nov 2026 | Design notes and diagram |
| 2. Development | Implement `EventTicketing` | 14–18 Nov 2026 | Compiling contract |
| 3. Testing | Positive, negative, boundary, state and access tests; coverage | 19–22 Nov 2026 | ≥ 50 tests, ≥ 85% coverage |
| 4. Security and remediation | Slither and manual review; fix and re-test | 23–26 Nov 2026 | Security report, remediation log |
| 5. Deployment, verification and report | Sepolia, Etherscan, report, README | 27–30 Nov 2026 | Verified URL, submitted ZIP |

## 10. References

[1] CSIT5760 Course Instructor, "CSIT5760 Blockchain and Smart Contracts — Individual Project Guidelines," Academic Year 2026–2027, 5 November 2026. [Accessed: 9 Nov. 2026].

[2] Solidity Team, "Solidity Documentation," v0.8.24, 2026. [Online]. Available: https://docs.soliditylang.org. [Accessed: 9 Nov. 2026].

[3] OpenZeppelin, "OpenZeppelin Contracts 5.x Documentation," 2026. [Online]. Available: https://docs.openzeppelin.com/contracts/5.x. [Accessed: 9 Nov. 2026].

[4] Nomic Foundation, "Hardhat Documentation," 2026. [Online]. Available: https://hardhat.org/docs. [Accessed: 9 Nov. 2026].

[5] Crytic, "Slither: Static Analyzer for Solidity," 2026. [Online]. Available: https://github.com/crytic/slither. [Accessed: 9 Nov. 2026].

[6] Etherscan, "Sepolia Testnet Explorer," 2026. [Online]. Available: https://sepolia.etherscan.io. [Accessed: 9 Nov. 2026].

[7] Eventbrite, "Organizer Fees," 2026. [Online]. Available: https://www.eventbrite.com/organizer/fees/. [Accessed: 9 Nov. 2026].

<!-- Acknowledge AI-assisted material here in the form required by the course policy (guidelines, Section 13). -->

## Proposal Submission Checklist

- ☐ Approximately 2–3 pages excluding references
- ☐ Scope is achievable before 30 November 2026
- ☐ At least three security risks are addressed
- ☐ Testing includes positive, negative, and boundary cases
- ☐ All third-party sources and libraries are acknowledged
- ☐ No private keys, seed phrases, or credentials are included

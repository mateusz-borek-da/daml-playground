## Problem Statement
Family wealth management involves multiple independent parties—such as family members, trust companies, and investment advisors—who need to collaborate on shared financial workflows while operating under different responsibilities and privacy requirements.

This project explores how Canton Network can enable these parties to coordinate around the same wealth-management process without sharing a single global dataset. Each participant should only receive the information required to perform its role.

The goal is to demonstrate how privacy-preserving, multi-party workflows can reduce unnecessary data exposure while maintaining coordinated, auditable financial processes.

## Actors in the system
| Actor                          | Role                                                 | Main responsibilities                                                        | Should see                                                                                 | Should **not** see                                                                                                                              |
| ------------------------------ | ---------------------------------------------------- | ---------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| **Founder / Family Principal** | Establishes and oversees the family wealth structure | Defines governance rules, appoints service providers, monitors family wealth | Assets, distributions, decisions of all actors           | internal process of RIA and trust-company |
| **Beneficiary**                | Person entitled to receive benefits or distributions | Requests distributions, tracks status, receives funds                        | Their own requests, entitlements, distribution status, permitted portfolio summaries (e.g. overall balance)       | Other beneficiaries’ requests, detailed portfolio holdings, RIA trading decisions                               |
| **Trust Company / Trustee**    | Independent fiduciary administering the trust        | Reviews requests, checks trust rules, approves or rejects distributions      | Beneficiary identity, request amount and purpose, trust rules    | Full portfolio strategy and details                |
| **RIA / Investment Advisor**   | Manages the investment portfolio                     | Manages assets and generates liquidity for approved distributions            | Portfolio holdings, required liquidity amount and timing  | Beneficiary identity, personal reason for a distribution             |

## Use case
### Distribution Request

The first use case demonstrates how a beneficiary, trust company, and investment advisor can coordinate a distribution while sharing only the information required by each participant.

* **Beneficiary** — requests a distribution from the family wealth structure.
* **Trust Company** — reviews the request and decides whether it can be approved.
* **RIA** — determines whether the required liquidity can be generated from the managed portfolio.

#### Flow

#### 1. Beneficiary creates a distribution request

The beneficiary submits a request containing information such as:

* requested amount;
* currency;
* purpose;
* requested payment date.


A `DistributionRequest` contract is created on Canton.

The contract is visible to:

```text
Beneficiary   ✓
Trust Company ✓
RIA           ✗
```
---

#### 2. Trust Company reviews the request

The Trust Company receives the `DistributionRequest` and verifies whether the request satisfies the applicable trust rules and governance requirements.

The contract exposes choices such as:

* Approve - If the request is approved, the Trust Company creates a separate `FundingRequirement`
* Reject - If the request is rejected, the workflow ends and the beneficiary can see the resulting rejected status.

---

#### 3. Funding requirement is created

The `FundingRequirement` represents the investment-side problem created by the approved distribution.

It does not contain beneficiary or purpouse related information.

The contract is visible to:

```text
Trust Company ✓
RIA           ✓
Beneficiary   ✗
```

The beneficiary does not need to know how the portfolio will be adjusted or whether the RIA is using cash, selling assets, or performing another liquidity operation.

---

#### 4. RIA processes the funding requirement

The RIA acts directly on the `FundingRequirement` contract.

Rather than creating an arbitrary off-ledger operation, the contract exposes explicit choices representing the possible outcomes:

* ProvideLiquidity - The RIA exercises this choice when it determines that sufficient liquidity can be generated. Exercising the choice produces a resulting contract such as FundingReady
* RejectFunding - The RIA exercises this choice when the requested liquidity cannot reasonably be generated and produces a new contract FundingRejected

---

#### 5. Trust Company receives the funding outcome

The Trust Company sees the result of the RIA decision. If the funding succeeds, it moves forward to the next section, otherwise it may:

* reject the distribution;
* request a lower amount;
* delay the distribution;
* initiate another funding attempt.

The beneficiary should see the resulting distribution-level status, but does not need to see the underlying `FundingRequirement`.

---

#### 6. Distribution becomes available

When the distribution has been approved and funding is available, the Trust Company creates a beneficiary-facing contract such as `DistributionAvailable`.

The contract is visible to:

```text
Beneficiary   ✓
Trust Company ✓
RIA           ✗
```

## Deployment

The application is deployed as a multi-validator Canton application. Each independent organization participating in the wealth-management workflow operates its own participant node.

## Network Topology

The initial deployment consists of three participant nodes:

```text
                         Canton Synchronizer
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                 │
              ▼                 ▼                 ▼

          Family Node     Trust Company Node      RIA Node
          Participant       Participant          Participant
              │                 │                 │
        Founder Party       Trustee Party        RIA Party
        Sophie Party
        Other beneficiary
        parties
              │                 │                 │
             PQS               PQS               PQS
              │                 │                 │
        PostgreSQL A      PostgreSQL B      PostgreSQL C
```

Each participant has:

* its own Canton participant/validator;
* its own Ledger API endpoint;
* its own PQS instance;
* its own PostgreSQL database;
* its own application credentials and keys.

The databases must not be shared between participants.

## Family Participant
It hosts parties Founder and Beneficiaries

Their visibility is still controlled at the Daml contract level. Hosting multiple family parties on the same participant means that the operator of the Family Participant is trusted with the data available to those parties.

## Trust Company Participant

It hosts the Canton party representing the trustee.

## RIA Participant

The RIA Participant belongs to the investment advisor responsible for managing the family's investment portfolio.

## Synchronizer

All three participant nodes should initially connect to **one common synchronizer**.

```text
                 Synchronizer

          /           |           \
         /            |            \

     Family       Trust Company      RIA
   Participant     Participant    Participant
```

## Future Extensions (TODO): 
* Multiple Synchronizers
* Multi-hosted parties
????
---
title: Intro to OP Stack
tags:
  - notes
---

# Background

Op stack powers Op Mainnet and Base
Ethereum's limited resources, specifically bandwidth, computation, and storage, constrain the number of transactions which can be processed on the network. Of the three resources, computation and storage are currently the most significant bottlenecks.
In a nutshell, an Optimistic Rollup utilizes Ethereum (or some other data availability layer) to host transaction data. Layer 2 nodes then execute a state transition function over this data. Users can propose the result of this off-chain execution to a smart contract on L1. A "fault proving" process can then demonstrate that a user's proposal is (or is not) valid.

## [Protocol Guarantees](https://specs.optimism.io/background.html#protocol-guarantees)

We strive to preserve three critical properties: liveness, validity, and availability. A protocol that can maintain these properties can, effectively, scale Ethereum without sacrificing security.

### [Liveness](https://specs.optimism.io/background.html#liveness)

Liveness is defined as the ability for any party to be able to extend the rollup chain by including a transaction within a bounded amount of time. It should not be possible for an actor to block the inclusion of any given transaction for more than this bounded time period. This bounded time period should also be acceptable such that inclusion is not just theoretically possible but practically useful.

### [Validity](https://specs.optimism.io/background.html#validity)

Validity is defined as the ability for any party to execute the rollup state transition function, subject to certain lower bound expectations for available computing and bandwidth resources. Validity is also extended to refer to the ability for a smart contract on Ethereum to be able to validate this state transition function economically.

### [Availability](https://specs.optimism.io/background.html#availability)

Availability is defined as the ability for any party to retrieve the inputs that are necessary to execute the rollup state transition function correctly. Availability is essentially an element of validity and is required to be able to guarantee validity in general.

There are three main actors that interact with the OP stack
1. Users
	-  Submit transactions through a Sequencer or by interacting with contracts on Ethereum
	- Query transaction data from interfaces operated by verifiers 
2. Sequencers
	-  Operates the role of the block producer, only limited to a single active sequencer
	-  Accepts transactions directly from users
	- observe deposit transactions on Ethereum (event)
	- consolidates both txn streams into ordered L2 blocks
	- submits info to L1s thats enough to reproduce those L2 blocks
	- provides real time access to pending L2 blocks that havent been confirmed on L1
3. Verifiers
	-  Download rollup data from L1 and sequencers
	-  use rollup data to execute L2 state transition function
	-  serve rollup data and computed L2 state info to users
	- submit assertions about the state of the L2 to a state contract on L1 
	- validate assertions made by other participants 
	- dispute invalid assertions made by other participants!
	
	
	![[OP_STACK.png]]

In short 
Users initiate the txn via interacting with the sequencers
Sequencers take the txns processes the txn offchain, orders them  and sends to the L1
Its the Verifiers job to verify data from the L1 and L2 to ensure uniform data across layers, basically a watchdog, also if a user wants to query the L2 state, its the job of the verifier to fetch the data from the sequencer

### Deposit - Transaction
Actors 
-  User
- Sequencer
- Ethereum
	- Optimism Portal (smart contract) 
	- Batch Inbox Address (not a smart contract)
In order to deposit the user has to do the following
1. Submit a deposit to the Optimism Portal (a smart contract) on Ethereum
2. The sequencer then listens for a deposit event
3. Once the event has been found, the sequencer then generates a deposit block
4. Now the user can send transactions, by submitting their txn to the sequencer
5. The sequencer takes the ordered txn and submit it to the Batch Inbox Address (smart contract ) on Ethereum

### Withdrawing
Actors
- User
- Proposers
- Sequencer
- Ethereum
	- Optimism Portal
	- Batch Inbox Address
	- DisputeGameFactory
		- Fault Dispute Game

In order to complete a withdrawal, the following steps has to be taken
1. User sends a withdrawal initialization txn to the Sequencer
2. The Sequencer submits the transaction batch to the Batch Inbox Address
3. The Proposers then submit an output proposal by calling the DisputeGameFactory
4. Then the Proposer generate a new "game", FaultDisputeGame from the DisputeGameFactory
5. The User then submit a withdrawal Proof to the Optimism Portal
6. The User then waits for finalization from the FaultDisputeGame
7. The User then submits a withdrawal finalization to the Optimism Portal
8. The Optimism Portal Checks the game validity of the FaultDisputeGame
9. The Optimism Portal then executes the withdrawal transaction 


Below is a simplified state-machine representation of the deposit and withdraw flow.  
Each block represents a protocol state and the condition under which a transition occurs.
# Deposit & Withdrawal State Machines

## Deposit Flow

```
Idle
  ↓ User submits deposit to OptimismPortal on Ethereum
DepositSubmitted
  ↓ Sequencer detects DepositEvent
DepositObserved
  ↓ Sequencer generates deposit block
L2DepositFinalized
  ↓ User sends transaction to Sequencer
UserTxPending
  ↓ Sequencer orders transactions and submits batch to BatchInbox on Ethereum
Completed ✓
```

### Detailed Steps

1. **Idle → DepositSubmitted**
    
    - User calls `submitDeposit()` on the OptimismPortal contract (Ethereum)
    - Deposit is recorded on L1
2. **DepositSubmitted → DepositObserved**
    
    - Sequencer listens for and detects the DepositEvent
    - Event must be sufficiently finalized before acting
3. **DepositObserved → L2DepositFinalized**
    
    - Sequencer generates a deposit block on L2
    - Funds are now represented on L2
4. **L2DepositFinalized → UserTxPending**
    
    - User sends their first L2 transaction to the Sequencer
    - Transaction enters the sequencer's queue
5. **UserTxPending → Completed**
    
    - Sequencer orders the transaction with others
    - Sequencer submits batch to BatchInbox on Ethereum
    - Deposit flow is complete; funds are usable on L2

---

## Withdrawal Flow

```
Idle
  ↓ User initiates withdrawal to Sequencer
WithdrawalInitiated
  ↓ Sequencer submits batch to BatchInbox on Ethereum
WithdrawalInL1Batch
  ↓ Proposer submits output proposal to L2OutputOracle
OutputProposed
  ↓ DisputeGameFactory creates FaultDisputeGame
GameCreated
  ↓ User submits withdrawal proof to OptimismPortal
ProofSubmitted
  ↓ Wait for game to finalize (security delay)
GameFinalized
  ↓ User submits finalization request to OptimismPortal
FinalizationRequested
  ↓ OptimismPortal verifies game validity
Completed ✓  OR  Reverted ✗
```

### Detailed Steps

1. **Idle → WithdrawalInitiated**
    
    - User sends withdrawal initialization to Sequencer
    - Withdrawal intent is recorded on L2
2. **WithdrawalInitiated → WithdrawalInL1Batch**
    
    - Sequencer includes withdrawal in a batch
    - Batch is submitted to BatchInbox on Ethereum
3. **WithdrawalInL1Batch → OutputProposed**
    
    - Proposer submits an output proposal to L2OutputOracle
    - This represents the L2 state containing the withdrawal
4. **OutputProposed → GameCreated**
    
    - DisputeGameFactory creates a FaultDisputeGame
    - This begins the challenge period for the withdrawal
5. **GameCreated → ProofSubmitted**
    
    - User submits withdrawal proof to OptimismPortal
    - Proof links their withdrawal to the output proposal
6. **ProofSubmitted → GameFinalized**
    
    - System waits for the dispute game to finalize
    - Security delay allows challengers to dispute invalid withdrawals
7. **GameFinalized → FinalizationRequested**
    
    - User requests withdrawal finalization from OptimismPortal
    - Final step before funds release
8. **FinalizationRequested → Completed or Reverted**
    
    - OptimismPortal verifies the game is valid
    - If valid: withdrawal executes, funds released on L1 ✓
    - If invalid: withdrawal reverts, no funds released ✗

---

## Key Concepts

### Actors

- **User**: Externally owned account initiating deposits or withdrawals
- **Sequencer**: Orders L2 transactions, produces batches, submits to L1
- **Proposer**: Submits L2 state outputs and initiates dispute games

### States

States represent logical protocol progress, not contract storage variables. Each state indicates where a deposit or withdrawal is in its lifecycle.

### Transitions

- **Event-driven**: Transitions occur when conditions are met, not on a timer
- **Monotonic**: Flow moves forward only (no rollbacks unless explicitly failed)
- **Gated**: Each transition requires specific actions and validations

### Chains

- **Ethereum (L1)**: Where deposits originate and withdrawals finalize
- **L2**: Where transactions execute and withdrawals initiate

### Security Features

- **WAIT states**: Represent security delays or liveness dependencies
- **Dispute games**: Allow challenges to invalid state proposals
- **Finalization checks**: Ensure only valid withdrawals complete

### Terminal States

- **Completed**: Successful end state (funds available)
- **Reverted**: Failed end state (withdrawal invalid, no further progress)

---

#### How to Read This Model 

> “Read each block as: _given the system is in this state, which actor is allowed to do what, and what state that action transitions the protocol into_.”

---

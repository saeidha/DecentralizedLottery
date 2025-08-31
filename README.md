# Decentralized Lottery/Raffle System - Solidity Smart Contract

## Overview
A trustless, transparent lottery system built on Ethereum using Chainlink VRF for provably fair random number generation. This contract allows users to participate in a lottery by purchasing tickets, with automatic winner selection and prize distribution.

## Technical Specifications

### Contract Features
- **Provably Fair Randomness**: Uses Chainlink VRF for verifiable randomness
- **Multi-Stage Lottery**: Implements state management (OPEN, CLOSED, CALCULATING_WINNER)
- **Automatic Payouts**: Distributes prizes without manual intervention
- **Historical Data**: Tracks previous winners and lottery history
- **Permissioned Administration**: Restricted functions for contract management

### State Variables
```solidity
enum LotteryState {
    OPEN,
    CLOSED,
    CALCULATING_WINNER
}

address[] public players;
address[] public previousWinners;
address public recentWinner;
uint256 public lotteryId;
uint256 public ticketPrice;
uint256 public maxPlayers;
uint256 public lotteryDuration;
uint256 public lotteryStartTime;
LotteryState public state;
address public owner;

// Chainlink VRF Variables
uint256 public vrfRequestId;
bytes32 public keyHash;
uint64 public subscriptionId;
uint32 public callbackGasLimit;
uint16 public requestConfirmations;
```

## Implementation Details

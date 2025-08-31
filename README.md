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

### Contract Structure

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.7;

import "@chainlink/contracts/src/v0.8/interfaces/VRFCoordinatorV2Interface.sol";
import "@chainlink/contracts/src/v0.8/VRFConsumerBaseV2.sol";
import "@chainlink/contracts/src/v0.8/ConfirmedOwner.sol";

contract DecentralizedLottery is VRFConsumerBaseV2, ConfirmedOwner {
    // State variables and events declaration here
    
    constructor(
        uint256 _ticketPrice,
        uint256 _maxPlayers,
        uint256 _lotteryDuration,
        address vrfCoordinator,
        bytes32 _keyHash,
        uint64 _subscriptionId,
        uint32 _callbackGasLimit,
        uint16 _requestConfirmations
    ) VRFConsumerBaseV2(vrfCoordinator) ConfirmedOwner(msg.sender) {
        // Initialization logic
    }
    
    // Function implementations
}
```

### Core Functions

#### Enter Lottery
```solidity
function enterLottery() public payable {
    require(state == LotteryState.OPEN, "Lottery is not open");
    require(msg.value == ticketPrice, "Incorrect ticket price");
    require(players.length < maxPlayers, "Maximum players reached");
    
    players.push(msg.sender);
    
    // Start lottery if first player and set timer
    if (players.length == 1) {
        lotteryStartTime = block.timestamp;
    }
    
    // Check if lottery should end
    if (block.timestamp >= lotteryStartTime + lotteryDuration || players.length == maxPlayers) {
        state = LotteryState.CLOSED;
        emit LotteryClosed(lotteryId, players.length);
    }
    
    emit PlayerEntered(msg.sender, lotteryId);
}
```

#### Request Random Winner
```solidity
function requestRandomWinner() public {
    require(state == LotteryState.CLOSED, "Lottery not closed");
    require(players.length > 0, "No players in lottery");
    
    state = LotteryState.CALCULATING_WINNER;
    
    // Request randomness from Chainlink VRF
    vrfRequestId = VRF_COORDINATOR.requestRandomWords(
        keyHash,
        subscriptionId,
        requestConfirmations,
        callbackGasLimit,
        1 // numWords
    );
    
    emit RandomWinnerRequested(lotteryId, vrfRequestId);
}
```

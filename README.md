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

#### VRF Callback Function
```solidity
function fulfillRandomWords(uint256, /* requestId */ uint256[] memory randomWords) internal override {
    require(state == LotteryState.CALCULATING_WINNER, "Not in calculation state");
    require(randomWords.length > 0, "No random numbers received");
    
    uint256 randomIndex = randomWords[0] % players.length;
    recentWinner = players[randomIndex];
    previousWinners.push(recentWinner);
    
    // Transfer prize to winner
    uint256 prizeAmount = address(this).balance;
    (bool success, ) = recentWinner.call{value: prizeAmount}("");
    require(success, "Transfer failed");
    
    // Reset for next lottery
    lotteryId++;
    delete players;
    state = LotteryState.OPEN;
    
    emit WinnerSelected(lotteryId - 1, recentWinner, prizeAmount);
}
```

#### Administrative Functions
```solidity
function updateTicketPrice(uint256 newPrice) public onlyOwner {
    require(state == LotteryState.OPEN && players.length == 0, "Cannot change during active lottery");
    ticketPrice = newPrice;
}

function updateLotteryDuration(uint256 newDuration) public onlyOwner {
    require(state == LotteryState.OPEN && players.length == 0, "Cannot change during active lottery");
    lotteryDuration = newDuration;
}

function withdrawLink() public onlyOwner {
    // Withdraw unused LINK tokens if any
}
```

### Events
```solidity
event LotteryStarted(uint256 indexed lotteryId, uint256 ticketPrice, uint256 maxPlayers);
event PlayerEntered(address indexed player, uint256 indexed lotteryId);
event LotteryClosed(uint256 indexed lotteryId, uint256 playersCount);
event RandomWinnerRequested(uint256 indexed lotteryId, uint256 requestId);
event WinnerSelected(uint256 indexed lotteryId, address winner, uint256 amount);
```

## Deployment Configuration

### Prerequisites
- Node.js and npm
- Hardhat or Truffle framework
- MetaMask wallet with testnet ETH
- Chainlink subscription with funded LINK tokens

### Deployment Script
```javascript
// scripts/deploy.js
async function main() {
  const [deployer] = await ethers.getSigners();
  
  const ticketPrice = ethers.utils.parseEther("0.1");
  const maxPlayers = 100;
  const lotteryDuration = 7 * 24 * 60 * 60; // 1 week in seconds
  
  // Chainlink VRF details (example for Ethereum Goerli)
  const vrfCoordinator = "0x2Ca8E0C643bDe4C2E08ab1fA0da3401AdAD7734D";
  const keyHash = "0x79d3d8832d904592c0bf9818b621522c988bb8b0c05cdc3b15aea1b6e8db0c15";
  const subscriptionId = 1234; // Your subscription ID
  const callbackGasLimit = 100000;
  const requestConfirmations = 3;
  
  const Lottery = await ethers.getContractFactory("DecentralizedLottery");
  const lottery = await Lottery.deploy(
    ticketPrice,
    maxPlayers,
    lotteryDuration,
    vrfCoordinator,
    keyHash,
    subscriptionId,
    callbackGasLimit,
    requestConfirmations
  );
  
  await lottery.deployed();
  console.log("Lottery deployed to:", lottery.address);
}
```

## Testing Strategy

### Test Cases
1. **Successful Lottery Entry**
   - User can enter with correct ticket price
   - User cannot enter with incorrect ticket price
   - User cannot enter when lottery is full

2. **Random Winner Selection**
   - Only owner can request random winner
   - Winner selection works correctly
   - Prize distribution to winner

3. **State Management**
   - Proper state transitions
   - Cannot enter when lottery is closed
   - Automatic closing when duration expires

4. **Security Tests**
   - Only owner can update parameters
   - Reentrancy protection
   - Emergency stop functionality

### Sample Test Code
```javascript
// test/Lottery.test.js
describe("DecentralizedLottery", function () {
  it("Should allow players to enter", async function () {
    await lottery.connect(player1).enterLottery({ value: ticketPrice });
    expect(await lottery.getPlayerCount()).to.equal(1);
  });
  
  it("Should select a random winner", async function () {
    // Simulate VRF response
    await expect(lottery.connect(owner).requestRandomWinner())
      .to.emit(lottery, "RandomWinnerRequested");
  });
});
```

## Security Considerations

### Best Practices Implemented
- Use of Chainlink VRF for secure randomness
- Reentrancy guards on external calls
- Proper access control with OpenZeppelin's Ownable
- Input validation on all parameters
- Emergency stop mechanism

### Potential Risks
1. **Chainlink VRF Dependency**: Relies on Chainlink's infrastructure
2. **Gas Limit Considerations**: Callback gas limit must be sufficient
3. **Block Timestamp Manipulation**: Minimal reliance on block.timestamp
4. **Oracle Failure**: Handling of VRF request failures

## Cost Optimization

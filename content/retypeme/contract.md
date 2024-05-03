---
title: "Contract"
---

{{< toc format=html >}}

## Test wallets

| Value    | Hash                                       |
|----------|--------------------------------------------|
| creator  | 0x93bfAD49c590A4E1f06A67EfbD4c90F94f501a85 |
| ap.test1 | 0xb4352Cf87739c3E494fd8fC405510dA02b61f66E |
| ap.test2 | 0x5057376A02B717e8B262C6fad5166c17e194D277 |

## Testnets

| chain            | block explorer                     | Chain ID  | RPC URL                                              |
|------------------|------------------------------------|-----------|------------------------------------------------------|
| blast-sepolia    | https://testnet.blastscan.io/      | 168587773 | https://blast-sepolia.infura.io/v3/{api-key}         |
| polygon-amoy     | https://www.oklink.com/amoy        | 80002     | https://polygon-amoy.infura.io/v3/{api-key}          |
| etherium-sepolia | https://sepolia.etherscan.io/      | 31337     | https://sepolia.infura.io/v3/{api-key}               |
| scroll-sepolia   | https://sepolia.scrollscan.com/    | 534351    | https://scroll-sepolia.core.chainstack.com/{api-key} |
| opbnb-testnet    | https://opbnb-testnet.bscscan.com/ | 5611      | ??                                                   |

## Contract

### Deployed Contracts

{{< tabs "release_versions" >}}
{{< tab "v2.0" >}}

#### Version 2.0 - 27.04.2024

| Network   | Contract Address                           |
|-----------|--------------------------------------------|
| Blast     | 0x993558c22ebe07c96e8f85d1ef4318c513abff0d |
| Scroll    | 0x078869dd68d019900098b5b1006951ea7b3f01f2 |
| Polygon   | 0x993558c22ebe07c96e8f85d1ef4318c513abff0d |
| BNB Chain | Not available                              |

{{< /tab >}}

{{< tab "v1.0" >}}

#### Version 1.0 - 01.03.2024

| Network   | Contract Address                           |
|-----------|--------------------------------------------|
| Blast     | 0xf813b4e5d34079ebcc59adf39a7782ad989891fe |
| Scroll    | Not available                              |
| Polygon   | 0xb3c33b58de859a5e06aff62c9d66319c256218da |
| BNB Chain | Not available                              |

{{< /tab >}}
{{< /tabs >}}

### Source code for latest contract version

{{< expand "v2.0"  >}}
```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.9;

interface IGamingContract {
    function deposit() external payable;

    function withdraw(uint256) external;

    function joinGame(uint256 _sessionId) external;

    function getBalance(address _user) external view returns (uint256);

    function isEnoughBalance(address _user) external view returns (bool);

    function getMinDeposit(address _user) external view returns (uint256);

    function endGame(uint256 _sessionId, address _winner) external;
}

contract GamingContract is IGamingContract {
    // declare constant default players count for duel mode
    uint256 public constant DUEL_PLAYERS_COUNT = 2;

    address public owner;
    uint256 public commissionRate;
    uint256 public fixedDepositAmount;
    uint256 public totalCommission;

    enum GameStatus {
        NOT_STARTED,
        STARTED,
        ENDED
    }

    struct GameSession {
        uint256 depositAmount;
        uint256 sessionId;
        GameStatus status;
        address[] players;
        address winner;
    }

    mapping(address => uint256) public inGameUserBalances;
    mapping(uint256 => GameSession) public gameSessions;

    event GameStarted(uint256 indexed sessionId);
    event GameEnded(uint256 indexed sessionId, address winner, uint256 winnings);

    modifier onlyOwner() {
        require(msg.sender == owner, "Not the owner");
        _;
    }

    constructor() {
        owner = msg.sender;
        commissionRate = 20;
        fixedDepositAmount = 0.001 ether;
    }

    function deposit() external payable {
        inGameUserBalances[msg.sender] += msg.value;
    }

    function getBalance(address _user) external view returns (uint256) {
        return inGameUserBalances[_user];
    }

    function getMinDeposit(address _user) external view returns (uint256) {
        // if there is no enough balance, show min amount to deposit to be able to play a game
        return fixedDepositAmount > inGameUserBalances[_user] ? fixedDepositAmount - inGameUserBalances[_user] : 0;
    }

    function isEnoughBalance(address _user) external view returns (bool) {
        return inGameUserBalances[_user] >= fixedDepositAmount;
    }

    // 0x4B20993Bc481177ec7E8f571ceCaE8A9e22C02d1
    function joinGame(uint256 _sessionId) external {
        // Check if the session already exists by checking a characteristic field, like sessionId
        if (gameSessions[_sessionId].sessionId == 0) {
            // This implies the session does not exist, so initialize it
            gameSessions[_sessionId].sessionId = _sessionId;
            gameSessions[_sessionId].depositAmount = 0; // Assuming it starts at 0 and increments with each deposit
            gameSessions[_sessionId].status = GameStatus.NOT_STARTED;
            gameSessions[_sessionId].players = new address[](0);
            gameSessions[_sessionId].winner = address(0);
        }
        // Proceed with existing checks
        require(
            gameSessions[_sessionId].players.length == 0 || gameSessions[_sessionId].players[0] != msg.sender,
            "Player already in the game"
        );
        require(gameSessions[_sessionId].players.length < DUEL_PLAYERS_COUNT, "Game session is full");
        require(gameSessions[_sessionId].status == GameStatus.NOT_STARTED, "Game session already started");
        require(inGameUserBalances[msg.sender] >= fixedDepositAmount, "Insufficient balance");

        // Lock sender balance
        inGameUserBalances[msg.sender] -= fixedDepositAmount;
        gameSessions[_sessionId].depositAmount += fixedDepositAmount;

        // Add the sender to the players list
        gameSessions[_sessionId].players.push(msg.sender);

        // If the session becomes full, start the game
        if (gameSessions[_sessionId].players.length == DUEL_PLAYERS_COUNT) {
            gameSessions[_sessionId].status = GameStatus.STARTED;
            emit GameStarted(_sessionId);
        }
    }

    function endGame(uint256 _sessionId, address _winner) external onlyOwner {
        GameSession storage session = gameSessions[_sessionId];
        // check if the game is started state
        require(session.status == GameStatus.STARTED, "Game not started or already ended");
        require(session.players.length == DUEL_PLAYERS_COUNT, "Wrong number of players in the game session");
        require(
            _winner == session.players[0] || _winner == session.players[1],
            "This address can't be the winner. It is not in the current game session."
        );
        // Logic to determine the winner and calculate winnings and commission
        // For simplicity, assuming winner is provided and winnings are total deposit amount minus commission
        // ADDED new variable (winnings) to make gas cheaper
        uint256 commission = (session.depositAmount * commissionRate) / 100;
        uint256 winnings = session.depositAmount - commission;
        // update winner balance
        inGameUserBalances[_winner] += winnings;
        totalCommission += commission;
        // update winner
        session.winner = _winner;
        // update session status
        session.status = GameStatus.ENDED;
        emit GameEnded(_sessionId, _winner, winnings);
    }

    // I think NO need to make two withdraw fns, because it will always get amount from FE after reading state. (there will be input
    // with desirable amount and button "Max", it will check max balance and pass it to the withdraw fn)
    function withdraw(uint256 _amount) external {
        address sender = msg.sender;
        require(inGameUserBalances[sender] >= _amount, "Insufficient balance");
        inGameUserBalances[sender] -= _amount;
        payable(sender).transfer(_amount);
    }

    function withdrawFundsFromContract() external onlyOwner {
        // we are withdrawing all the balance. will be deleted in future versions!
        require(address(this).balance > 0, "No funds to withdraw");
        payable(owner).transfer(address(this).balance);
    }

    function withdrawCommissionFromContract() external onlyOwner {
        // withdrawing only total commission
        require(totalCommission > 0, "No commission to withdraw");
        totalCommission = 0; // MODIFY FIRST
        // payable(owner).transfer(totalCommission); // THEN TRANSFER
        payable(owner).transfer(totalCommission); // THEN TRANSFER
    }

    function changeCommissionRate(uint256 _newCommissionRate) external onlyOwner {
        require(_newCommissionRate >= 0 && _newCommissionRate < 100, "Enter valid number");
        commissionRate = _newCommissionRate;
    }

    function changeFixedDepositAmount(uint256 _newFixedDepositAmount) external onlyOwner {
        require(_newFixedDepositAmount > 0, "Enter valid number");
        fixedDepositAmount = _newFixedDepositAmount;
    }
}
```
{{< /expand >}}

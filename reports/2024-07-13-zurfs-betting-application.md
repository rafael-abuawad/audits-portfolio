# Zurf Betting Application

## Table of contents

1. **[Zurf Betting Application](#zurf-betting-application)**
2. **[Audit Report](#audit-report)**
	* [Prepared by: Rafael Abuawad](#prepared-by-rafael-abuawad)
	* [Lead Auditor: Rafael Abuawad](#lead-auditor-rafael-abuawad)
3. **[About Rafael Abuawad](#about-rafael-abuawad)**
4. **[Disclaimer](#disclaimer)**
5. **[Audit Details](#audit-details)**
	* [In Scope](#in-scope)
	* [Solc Version](#solc-version)
	* [Chain(s) to deploy the contract to](#chains-to-deploy-the-contract-to)
6. **[Protocol Summary](#protocol-summary)**
7. **[Roles](#roles)**
	* [Bet Creator](#bet-creator)
	* [Bettors](#bettors)
8. **[Risk Classification](#risk-classification)**
9. **[Executive Summary](#executive-summary)**
10. **[Issues found](#issues-found)**
11. **[Findings](#findings)**
	* **[High](#high)**
		+ [[H-1] `FootballBetting::minimumBetAmount` can be overwritten by anyone creating a new bet](#h-1-footballbettingminimumbetamount-can-be-overwritten-by-anyone-creating-a-new-bet)
		+ [[H-2] If `FootballBetting::distributeWinnings` is called with a winner that has no bets in its favor the contract reverts](#h-2-if-footballbettingdistributewinnings-is-called-with-a-winner-that-has-no-bets-in-its-favor-the-contract-reverts)
	* **[Medium](#medium)**
		+ [[M-1] Place Bet On Option methods could be simplified using 1 single method](#m-1-place-bet-on-option-methods-could-be-simplified-using-1-single-method)
	* **[Low](#low)**
		+ [[L-1] `IERC20` interface should be defined on a separate file](#l-1-ierc20-interface-should-be-defined-on-a-separate-file)
		+ [[L-2] `FootballBetting::BONSAI_TOKEN_ADDRESS` is just used to set the `FootballBetting::bonsaiToken` and can be removed](#l-2-footballbettingbonsai_token_address-is-just-used-to-set-the-footballbettingbonstoken-and-can-be-removed)
		+ [[L-3] `FootballBetting::bonsaiToken` should be set as an immutable variable](#l-3-footballbettingbonstoken-should-be-set-as-an-immutable-variable)
		+ [[L-4] Typo on the `FootballBeting::placeBetOnoption1` and `FootballBetting::placeBetOnoption2` function signature](#l-4-typo-on-the-footballbettingplacebetonoption1-and-footballbettingplacebetonoption2-function-signature)
		+ [[L-5] `FootballBetting::openBetting` does not emit any event](#l-5-footballbettingopenbetting-does-not-emit-any-event)
		+ [[L-6] There are 12 instances of the word `require` which is less gas efficient](#l-6-there-are-12-instances-of-the-word-require-which-is-less-gas-efficient)

## Audit Report

Prepared by: Rafael Abuawad

Lead Auditor: 
- [Rafael Abuawad](https://github.com/rafael-abuawad/)

## About Rafael Abuawad

Hi, I'm Rafael, I am a highly skilled software developer specializing in the exciting realm of Blockchain and Decentralized Applications. With over 6 years of professional experience, I have successfully crafted and delivered 30+ projects for diverse clients, ranging from individuals to small and large teams. My expertise lies in harnessing the power of cutting-edge technology. You can learn more about me [here](https://github.com/rafael-abuawad/).

## Disclaimer

I (Rafael Abuawad) made all the effort to find as many vulnerabilities in the code in the given time period. Still, I hold no responsibility for the findings provided in this document. A security audit by myself is not an endorsement of the underlying business or product. The audit was time-boxed and the review of the code was solely on the security aspects of the solidity implementation of the contracts.

## Audit Details
- In Scope:
```
./contracts/
└── FootballBetting.sol
```
- Solc Version: 0.8.24
- Chain(s) to deploy the contract to: Polygon


# Protocol Summary 

A simple betting smart contract designed to distribute tokens based on two (2) choices. Although I did lack the intentions of the smart contract, specific details, and better documentation in general. Since this was just a POC, that is not too much of a deal, but in the future, I highly recommend following development best practices, like having good docs, specs, well-documented roles, and a clear intent on how the smart contract is going to be used.

## Roles

- Bet Creator: The address that initiates a bet.
- Bettors: Addresses that enter & and participate in a bet.


# Risk Classification

|                 |        | Impact |        |     |
| --------------- | ------ | ------ | ------ | --- |
|                 |        | High   | Medium | Low |
|                 | High   | H      | H/M    | M   |
| **Likelihood**  | Medium | H/M    | M      | M/L |
|                 | Low    | M      | M/L    | L   |

# Executive Summary

## Issues found

| Severity          | Number of issues found |
| ----------------- | ---------------------- |
| High              | 2                      |
| Medium            | 1                      |
| Low               | 6                      |
| Total             | 9                      |

# Findings

## High 

### [H-1]  `FootballBetting::minimumBetAmount` can be overwritten by anyone creating a new bet, affecting ongoing bets as well

**Description:** The `FootballBetting::minimumBetAmount` variable is used to confirm that the amount in *BONSAI* is correct, but it is a global variable, which means that the same `FootballBetting::minimumBetAmount` is used for every bet. This can be used maliciously, intentionally or unintentionally by someone creating a new bet.

**Impact:** Someone could lower or increase the bet amount just by creating a disposable/dummy bet. Making the original `FootballBetting::minimumBetAmount` useless, and disrupting the expetation of the Creator and the Bettors.

**Proof of Concept:** 

```bash
$ ape test --network ::foundry -k "test_minimum_bet_amount_overwrite"
```

```python
import random
import ape

def test_minimum_bet_amount_overwrite(football_betting, bonsai, creator, users):
    bet_id = "1" 
    options = ["Team A", "Team B"]
    minimum_bet_amount = int(10e18) # 10 MOCK

    # Create a bet, with a minimum bet amount of 10 MOCK
    football_betting.createBetInstance(bet_id, options[0], options[1], minimum_bet_amount, sender=creator)

    # People can enter without any problems
    for user in users:
        bonsai.approve(football_betting, minimum_bet_amount, sender=user)
        football_betting.placeBetOnoption1(options[0], minimum_bet_amount, sender=user)

    # A random user creates a new bet, with a minimum bet amount of 120 MOCK
    new_minimum_bet_amount = int(120e18) # 120 MOCK
    new_bet_id = "2" 
    random_user = random.choice(users)
    football_betting.createBetInstance(new_bet_id, options[0], options[1], new_minimum_bet_amount, sender=random_user)

    # People cannot enter Bet ID #1 because the Minimum bet is now 120 MOCK
    for user in users:
        with ape.reverts("revert: Bet amount is less than minimum"):
            football_betting.placeBetOnoption1(bet_id, minimum_bet_amount, sender=user)

```

**Recommended Mitigation:** The `minimumBetAmount` should be part of the Bet instance.
```diff
-     uint256 private minimumBetAmount;
```

```diff
    struct BetInstance {
        string id;
        string option1;
        string option2;
        uint256 totalAmountOption1;
        uint256 totalAmountOption2;
+        uint256 minimumBetAmount;
        mapping(address => Bet) betsoption1;
        mapping(address => Bet) betsoption2;
        address[] bettorsoption1;
        address[] bettorsoption2;
        address creator;
        bool isClosed;
        bool isResolved;
    }
```

```diff
    function createBetInstance(string calldata _betId, string calldata _option1, string calldata _option2, uint256 _minimumBetAmount) external {
        require(bytes(betInstances[_betId].id).length == 0, "Bet instance with this ID already exists");

        BetInstance storage newInstance = betInstances[_betId];
        newInstance.id = _betId;
        newInstance.option1 = _option1;
        newInstance.option2 = _option2;
        newInstance.creator = msg.sender;
-       minimumBetAmount = _minimumBetAmount;
+       newInstance.minimumBetAmount = _minimumBetAmount;
        emit BetInstanceCreated(msg.sender, _betId, _option1, _option2, _minimumBetAmount);
    }
```

```diff
    function placeBetOnoption1(string calldata _betId, uint256 _amount) external {
        BetInstance storage betInstance = betInstances[_betId];
        require(!betInstance.isClosed, "Betting is closed");
-       require(_amount >= minimumBetAmount, "Bet amount is less than minimum");
+       require(_amount >= betInstance.minimumBetAmount, "Bet amount is less than minimum");
        require(bonsaiToken.transferFrom(msg.sender, address(this), _amount), "Transfer failed");

        betInstance.totalAmountOption1 += _amount;
        betInstance.betsoption1[msg.sender] = Bet(msg.sender, _amount);
        betInstance.bettorsoption1.push(msg.sender);
        
        emit BetPlaced(_betId, msg.sender, _amount, 1);
    }

    function placeBetOnoption2(string calldata _betId, uint256 _amount) external {
        BetInstance storage betInstance = betInstances[_betId];
        require(!betInstance.isClosed, "Betting is closed");
-       require(_amount >= minimumBetAmount, "Bet amount is less than minimum");
+       require(_amount >= betInstance.minimumBetAmount, "Bet amount is less than minimum");
        require(bonsaiToken.transferFrom(msg.sender, address(this), _amount), "Transfer failed");

        betInstance.totalAmountOption2 += _amount;
        betInstance.betsoption2[msg.sender] = Bet(msg.sender, _amount);
        betInstance.bettorsoption2.push(msg.sender);

        emit BetPlaced(_betId, msg.sender, _amount, 2);
    }
```

### [H-2] If `FootballBetting::distributeWinnings` is called with a winner that has no bets in its favor the contract reverts, giving anyone the opportunity to place a minimum bet on that winner and taking the entire bet winnings for himself/herself.


**Description:** `FootballBetting::distributeWinnings` has a check in place to see if the winner has bets placed in its favor, if this is not the case the function will revert, but this opens up an opportunity for an attacker or for a malicious actor to place a bet on the winning side just as it reverts and taking the entire price pot for himself or herself.

**Impact:** This exploit can be used to take the last had on a bet, although this could be mitigated by calling `Football::closeBetting` on that bet, the fact that a smart actor could use this to win more money a security risk. 

**Proof of Concept:** 

```bash
$ ape test --network ::foundry -k "test_taking_the_entire_price_for_attacker"
```

```python
def test_taking_the_entire_price_for_attacker(football_betting, bonsai, creator, users):
    bet_id = "1" 
    options = ["Team A", "Team B"]
    minimum_bet_amount = int(10e18) # 10 MOCK

    # Create a bet, with a minimum bet amount of 10 MOCK
    football_betting.createBetInstance(bet_id, options[0], options[1], minimum_bet_amount, sender=creator)

    # People can enter without any problems
    for user in users:
        bonsai.approve(football_betting, minimum_bet_amount, sender=user)
        football_betting.placeBetOnoption1(bet_id, minimum_bet_amount, sender=user)

    # Creator closes the bet
    football_betting.closeBetting(bet_id, sender=creator)
    
    # The creator declares Team B is the winner, and the transaction reverts
    with ape.reverts("revert: No bets on Team 2"):
        football_betting.distributeWinnings(bet_id, 2, sender=creator)
    
    # Creator re-opens the bet
    football_betting.openBetting(bet_id, sender=creator)
    
    # He bets on the winning team (AKA Team 2)
    bonsai.approve(football_betting, minimum_bet_amount, sender=creator)
    football_betting.placeBetOnoption2(bet_id, minimum_bet_amount, sender=creator)

    # Creator intial MOCK balance
    initial_balance = bonsai.balanceOf(creator)
    pot = bonsai.balanceOf(football_betting)
    pot = int(pot - pot*0.01)

    # Creator closes the bet
    football_betting.closeBetting(bet_id, sender=creator)
    
    # Distribtes all earnings to himself (minus the fee)
    football_betting.distributeWinnings(bet_id, 2, sender=creator)
    
    new_balance = bonsai.balanceOf(creator)
    assert new_balance == (initial_balance + pot)
```

**Recommended Mitigation:** I would personally recommend distributing back the BONSAI tokens to all participants if no winner was found. This method can still charge fees.

Add a new private method for returning fees:
```cs
function _noWinner(string calldata _betId, uint8 _winningTeam) private {
    BetInstance storage betInstance = betInstances[_betId];
    uint256 totalFees = 0;

    if (_winningTeam == 1 && betInstance.totalAmountOption1 == 0) {
        address[] memory players = betInstance.bettorsoption2;
        for (uint256 i = 0; i < players.length; i++) {
            address player = players[i];
            uint256 betAmount = betInstance.betsoption2[player].amount;
            uint256 creatorFee = betAmount / 100;
            totalFees += creatorFee;

            require(bonsaiToken.transfer(player, betAmount - creatorFee), "Winner transfer failed");
        }
    }

    if (_winningTeam == 2 && betInstance.totalAmountOption2 == 0) {
        address[] memory players = betInstance.bettorsoption1;
        for (uint256 i = 0; i < players.length; i++) {
            address player = players[i];
            uint256 betAmount = betInstance.betsoption1[player].amount;
            uint256 creatorFee = betAmount / 100;
            totalFees += creatorFee;

            require(bonsaiToken.transfer(player, betAmount - creatorFee), "Winner transfer failed");
        }
    }

    require(bonsaiToken.transfer(FEE_ADDRESS, totalFees), "Winner transfer failed");
}
```

Use it here, inside `FootballBettings::distributeWinnings`:
```diff
function distributeWinnings(string calldata _betId, uint8 _winningTeam) external onlyCreator(_betId) {
    require(_winningTeam == 1 || _winningTeam == 2, "Invalid winning team");

    BetInstance storage betInstance = betInstances[_betId];
    require(betInstance.isClosed && !betInstance.isResolved, "Betting must be closed and not resolved");

    uint256 totalAmountToDistribute = betInstance.totalAmountOption1 + betInstance.totalAmountOption2;
    uint256 totalAmountWinningTeam;

+   if (betInstance.totalAmountOption1 == 0 || betInstance.totalAmountOption2 == 0) {
+       _noWinner(_betId, _winningTeam);
+       betInstance.isResolved = true;
+       emit BettingResolved(_betId, _winningTeam);
+       return;
+   }

    // Determinar el total apostado en el equipo ganador
    if (_winningTeam == 1) {
-       require(betInstance.totalAmountOption1 > 0, "No bets on Team 1");
        totalAmountWinningTeam = betInstance.totalAmountOption1;
    } else {
-       require(betInstance.totalAmountOption2 > 0, "No bets on Team 2");
        totalAmountWinningTeam = betInstance.totalAmountOption2;
    }

    ...
}
```

This works as expected:

```bash
$ ape test --network ::foundry -k "test_place_bets"
```

```python
def test_place_bets(football_betting, bonsai, creator, users):
    bet_id = "1" 
    options = ["Team A", "Team B"]
    minimum_bet_amount = int(100_000e18) # 100.000 MOCK

    # Create bet, with minimum bet amount of 100.000 MOCK
    football_betting.createBetInstance(bet_id, options[0], options[1], minimum_bet_amount, sender=creator)

    # We want to keep track of the user address and their choice
    balances = {}
    choices = {}

    # People can enter without any problems
    for user in users:
        bonsai.approve(football_betting, minimum_bet_amount, sender=user)

        option = random.choice(options)
        if option == "Team A":
            football_betting.placeBetOnoption1(bet_id, minimum_bet_amount, sender=user)
        elif option == "Team B": 
            football_betting.placeBetOnoption2(bet_id, minimum_bet_amount, sender=user)

        choices[str(user)] = option
        balances[str(user)] = bonsai.balanceOf(user)

    # Creator closes the bet
    football_betting.closeBetting(bet_id, sender=creator)

    # And the creator declares Team B is the winner
    football_betting.distributeWinnings(bet_id, 2, sender=creator)

    # Users on the winning side should see a balance increase, and
    # users on the losing side should see the same balance.
    for user in users:
        option = choices[str(user)]
        initial_balance = balances[str(user)]

        if option == "Team A":
            assert bonsai.balanceOf(user) == initial_balance
        elif option == "Team B": 
            assert bonsai.balanceOf(user) > initial_balance
```

## Medium 

### [M-1] Place Bet On Option methods could be simplified using 1 single method.

**Description:** There is unnecessary repetition on these 2 functions, most of the logic can be abstracted away into one single private method and the developer can call that instead.

**Recommended Mitigation:** 

```diff
+   function _placeBetOnOption(string calldata _betId, uint256 _amount, uint256 _option) private {
+       BetInstance storage betInstance = betInstances[_betId];
+       require(!betInstance.isClosed, "Betting is closed");
+       require(_amount >= minimumBetAmount, "Bet amount is less than minimum");
+       require(bonsaiToken.transferFrom(msg.sender, address(this), _amount), "Transfer failed");

+       if (_option == 1) {
+           betInstance.totalAmountOption1 += _amount;
+           betInstance.betsoption1[msg.sender] = Bet(msg.sender, _amount);
+           betInstance.bettorsoption1.push(msg.sender);
+           emit BetPlaced(_betId, msg.sender, _amount, 1);
+           return;
+       }

+       if (_option == 2) {
+           betInstance.totalAmountOption2 += _amount;
+           betInstance.betsoption2[msg.sender] = Bet(msg.sender, _amount);
+           betInstance.bettorsoption2.push(msg.sender);
+           emit BetPlaced(_betId, msg.sender, _amount, 2);
+           return;
+       }
+   }

    function placeBetOnoption1(string calldata _betId, uint256 _amount) external {
-       BetInstance storage betInstance = betInstances[_betId];
-       require(!betInstance.isClosed, "Betting is closed");
-       require(_amount >= minimumBetAmount, "Bet amount is less than minimum");
-       require(bonsaiToken.transferFrom(msg.sender, address(this), _amount), "Transfer failed");

-       betInstance.totalAmountOption1 += _amount;
-       betInstance.betsoption1[msg.sender] = Bet(msg.sender, _amount);
-       betInstance.bettorsoption1.push(msg.sender);
-       
-       emit BetPlaced(_betId, msg.sender, _amount, 1);
+       _placeBetOnOption(_betId, _amount, 1);
    }

    function placeBetOnoption2(string calldata _betId, uint256 _amount) external {
-       BetInstance storage betInstance = betInstances[_betId];
-       require(!betInstance.isClosed, "Betting is closed");
-       require(_amount >= minimumBetAmount, "Bet amount is less than minimum");
-       require(bonsaiToken.transferFrom(msg.sender, address(this), _amount), "Transfer failed");

-       betInstance.totalAmountOption2 += _amount;
-       betInstance.betsoption2[msg.sender] = Bet(msg.sender, _amount);
-       betInstance.bettorsoption2.push(msg.sender);

-       emit BetPlaced(_betId, msg.sender, _amount, 2);
+       _placeBetOnOption(_betId, _amount, 2);
    }


```

## Low

### [L-1] `IERC20` interface should be defined on a separate file

**Description:** The `IERC20` interface is defined on the same file as `FootballBetting`, there are 2 possible solutions to this, you could use an external library, or (to keep things simple) you could define the same interface in a different file and import that on `FootballBetting`.

**Recommended Mitigation:** 

```
./contracts/
└──./interfaces/
    └── IERC20.sol
    FootballBetting.sol
```

```diff
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

interface IERC20 {
    function transferFrom(address sender, address recipient, uint256 amount) external returns (bool);
    function transfer(address recipient, uint256 amount) external returns (bool);
    function balanceOf(address account) external view returns (uint256);
}
```

```diff
- interface IERC20 {
-   function transferFrom(address sender, address recipient, uint256 amount) external returns (bool);
-   function transfer(address recipient, uint256 amount) external returns (bool);
-   function balanceOf(address account) external view returns (uint256);
- }

+ import {IERC20} from "./interfaces/IERC20.sol";
```

### [L-2] `FootballBetting::BONSAI_TOKEN_ADDRESS` is just used to set the `FootballBetting::bonsaiToken` and can be removed

**Description:** The `FootballBetting::BONSAI_TOKEN_ADDRESS` constant is used only once to set the `FootballBetting::bonsaiToken`, is taking extra memory space and should be removed.

**Recommended Mitigation:** 

```diff
- address private immutable BONSAI_TOKEN_ADDRESS;

+ constructor(address _bonsaiTokenAddress) {
+    bonsaiToken = IERC20(_bonsaiTokenAddress);
+ }

```

### [L-3] `FootballBetting::bonsaiToken` should be set as an immutable variable

**Description:** The `FootballBetting::bonsaiToken` can be set to an immutable variable to save on gas costs.

**Recommended Mitigation:** 

```diff
- IERC20 private bonsaiToken;
+ IERC20 private immutable bonsaiToken;
```

### [L-4] Typo on the `FootballBeting::placeBetOnoption1` and `FootballBetting::placeBetOnoption2` function signature

**Description:** `FootballBeting::placeBetOnoption1` and `FootballBetting::placeBetOnoption2` should be `FootballBeting::placeBetOnOption1` and `FootballBetting::placeBetOnOption2`

**Recommended Mitigation:** 

```diff
-   function placeBetOnoption1(string calldata _betId, uint256 _amount) external {
+   function placeBetOnOption1(string calldata _betId, uint256 _amount) external {
        BetInstance storage betInstance = betInstances[_betId];
        require(!betInstance.isClosed, "Betting is closed");
        require(_amount >= minimumBetAmount, "Bet amount is less than minimum");
        require(bonsaiToken.transferFrom(msg.sender, address(this), _amount), "Transfer failed");

        betInstance.totalAmountOption1 += _amount;
        betInstance.betsoption1[msg.sender] = Bet(msg.sender, _amount);
        betInstance.bettorsoption1.push(msg.sender);
        
        emit BetPlaced(_betId, msg.sender, _amount, 1);
    }

-   function placeBetOnoption2(string calldata _betId, uint256 _amount) external {
+   function placeBetOnOption2(string calldata _betId, uint256 _amount) external {
        BetInstance storage betInstance = betInstances[_betId];
        require(!betInstance.isClosed, "Betting is closed");
        require(_amount >= minimumBetAmount, "Bet amount is less than minimum");
        require(bonsaiToken.transferFrom(msg.sender, address(this), _amount), "Transfer failed");

        betInstance.totalAmountOption2 += _amount;
        betInstance.betsoption2[msg.sender] = Bet(msg.sender, _amount);
        betInstance.bettorsoption2.push(msg.sender);

        emit BetPlaced(_betId, msg.sender, _amount, 2);
    }
```

### [L-5] `FootballBetting::openBetting` does not emit any event

**Description:** The `FootballBetting::bonsaiToken` can be set to an immutable variable to save on gas costs.

**Recommended Mitigation:** 

```diff
+   event BettingOpen(string indexed betId);

    function openBetting(string calldata _betId) external onlyCreator(_betId) {
        BetInstance storage betInstance = betInstances[_betId];
        betInstance.isClosed = false;

+       emit BettingOpen(_betId);
    }
```

### [L-6] There are 12 instances of the word `require` which is less gas efficient, the developer should consider using `if (...) { revert CustomError() }` instead. 

**Description:** Using the `require` keyword is not as efficient as using a custom error inside an if statement in Solidity. [You can learn more about it here.](https://www.alchemy.com/overviews/solidity-require)

**Recommended Mitigation:** The refactor would involve creating multiple custom errors and changing the `require` statements with `if (...) { revert CustomError() }` ones.

```diff
+ error FootballBetting__NotAuthorized();

modifier onlyCreator(string memory _betId) {
-   require(msg.sender == betInstances[_betId].creator, "Not authorized");
+   if(msg.sender!= betInstances[_betId].creator) {
+       revert FootballBetting__NotAuthorized();
+   }
    _;
}
```
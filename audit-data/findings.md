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
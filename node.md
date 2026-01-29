duplicate addresses are not allowed in a singile raffle.
 the entrancefee use require: they can just use custom to save gas

 they use for loop it can be gas expensive if the list is long: will be inconvinent to use
   why not use Mapping instead

 the check for dublicate also use loop litralyy gas expensive: use mapping


 t
 Refund: players can request for fund while winner is been selected

 their is a probaily of user requested rund and be still picked as weinner



the GetActive player can be gas expensive: use mapping instead



winner selection 
 anyone can call the function


the winner index selection is not random it is priditable a MEV can easliy predict as so others: i will drop a exmp



winner that is been  selected  can be address 0 since users can request for refund and it will be replaced with address 0


 the tokenId is pridictable  which can cause the minting to revert duplicat id also totalSupply is not defined nor in the ERC721


 the _exits is not defined or imported




the rarity selection is not actionally radom. it can be pridictable

 the winner address that is going to be paid is not payable which will cause it to revert



 the withraw: if someone force or send even 1wei the owner wont be able to withdraw: logical erorr 
 the feeAddress  in the withdraw will rever cause it lack payble





 the getActive players will be gas expensive if playesr get bigger: use mapping instead

isactive players  will also get gas expensive: use mapping instead



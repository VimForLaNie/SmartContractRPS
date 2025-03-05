# SmartContractRPS
## Rock Paper Scissor Lizard and Spock on ETH smart contract
![RPSLS](https://static.wikia.nocookie.net/bigbangtheory/images/7/7d/RPSLS.png/revision/latest?cb=20120822205915)

This game logic can be simply implemented using modulo arithmetic by representing each player choice with an interger modulo 5.

Given states of two players to be $A$ and $B$ :

The player with state $A$ will win iff $A + 1 \equiv B \pmod 5$ or $A + 3 \equiv B  \pmod 5$

This smart contract consist of 3 modules :
- `RPS.sol` main game
- `CommitReveal.sol` to help hide the players choices and avoid front-running
- `TimeUnit.sol` to help with dealing time

## How it works?
1. Each player send 1 ETH to participate in this game. If only 1 player send in ETH and at least 3 minutes has passed, the player can withdraw the ETH by calling `abort` function.
2. Each player make their choices :
   - `00` for Spock
   - `01` for Lizard
   - `02` for Scissor
   - `03` for Paper
   - `04` for Rock
   
   Then each player has to generate a random hex string, 30 characters long and append their choices to the string and send the result using `input` function.

**If the second player would not make any choice, the first player can `abort` the game after 3 minutes has passed.**
 
 3. Each player reveal thier choices (this function available only if both players have make their choices) using `reveal` funtion and passing the string from step 2 as input.
 4. After both players reveal their choices, the contract will pay the player accordingly :
   - Win 2 ETH
   - Draw 1 ETH each

## Behind the Scene
### Commit & Reveal
```Solidity
function commit(address user,bytes32 dataHash) public {
    commits[user].commit = dataHash;
    commits[user].block = uint64(block.number);
    commits[user].revealed = false;
    emit CommitHash(user,commits[user].commit,commits[user].block);
}
```
```Solidity
function reveal(address user,bytes32 revealHash) public {
    //make sure it hasn't been revealed yet and set it to revealed
    require(commits[user].revealed==false,"CommitReveal::reveal: Already revealed");
    //require that they can produce the committed hash
    require(getHash(revealHash)==commits[user].commit,"CommitReveal::reveal: Revealed hash does not match commit");
    //require that the block number is greater than the original block
    require(uint64(block.number)>commits[user].block,"CommitReveal::reveal: Reveal and commit happened on the same block");
    //require that no more than 250 blocks have passed
    require(uint64(block.number)<=commits[user].block+250,"CommitReveal::reveal: Revealed too late");
    //get the hash of the block that happened after they committed
    bytes32 blockHash = blockhash(commits[user].block);
    //hash that with their reveal that so miner shouldn't know and mod it with some max number you want
    uint random = uint(keccak256(abi.encodePacked(blockHash,revealHash)))%max;
    //successfully revealed
    commits[user].revealed=true;
    emit RevealHash(user,revealHash,random);
  }
```
I modify the commit and reveal function to allow passing the message sender to the function since `msg.sender` in the context of a sub-module will be the main-module address instead of the message sender.

### Preventing Locking ETH inside the contract
I implement the `abort` function to help dealing with cases that can locked your ETH inside the contract
```Solidity
function abort() public {
    // you are very lonely
    if(numPlayer == 1 && timeUnit.elapsedMinutes() > 3 minutes){
        address payable account0 = payable(players[0]);
        account0.transfer(reward); 
        reward = 0;
    }
    // the other player won't make their choice
    else if (numPlayer == 2 && numInput == 1 && timeUnit.elapsedMinutes() > 3 minutes) {
        address payable account0 = payable(players[0]);
        address payable account1 = payable(players[1]);
        account0.transfer(reward / 2);
        account1.transfer(reward / 2); 
        reward = 0;
    }
}
```
### Deciding Winner
The choice that a player made can be obtain by extracting the last digit of the hex string ( `& 0xf` works fine but since the state of the game is defined with 2 digit, `& 0xff` is used instead).
```Solidity
function reveal(bytes32 hash) public {
    require(numInput == 2,"All players must make their choice before reveal");
    commitReveal.reveal(msg.sender,hash);
    //obtain the choice which is the last character of the string
    player_choice[msg.sender] = uint256(hash) & 0xff;
    numRevealed++;
    if (numRevealed == 2) {
        _checkWinnerAndPay();
    }
}
```
If the both player revealed their choices, we call `_checkWinnerAndPay`. This is mostly the same as [https://github.com/parujr/RPS/blob/main/RPS.sol]. 
```Solidity
function _checkWinnerAndPay() private {
        // players choices
        uint p0Choice = player_choice[players[0]];
        uint p1Choice = player_choice[players[1]];
        // player's accounts
        address payable account0 = payable(players[0]);
        address payable account1 = payable(players[1]);
        // rock paper scissor lizard spock
        if ((p0Choice + 1) % 5 == p1Choice || (p0Choice + 3) % 5 == p1Choice) {
            // to pay player[1]
            account1.transfer(reward);
        }
        else if ((p1Choice + 1) % 5 == p0Choice || (p1Choice + 3) % 5 == p0Choice) {
            // to pay player[0]
            account0.transfer(reward);    
        }
        else {
            // to split reward
            account0.transfer(reward / 2);
            account1.transfer(reward / 2);
        }
        reward = 0;
    }
```

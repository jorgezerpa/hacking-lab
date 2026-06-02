# Easy Money Makes You Part of a Big Scam

Disclaimer: This analysis covers a fully malicious smart contract. It is published exclusively for educational purposes to help users identify scams. Do not interact with this contract. Anyone choosing to interact with this code does so entirely at their own risk and assumes full liability for any losses.

I have seen these kinds of contracts multiple times. Sometimes they have +40 ETH, other times only 1 ETH, but the stealing mechanism is always the same, and the scam isn't very obvious to normal crypto users.

### First, let's see the contract code:

```Solidity
pragma solidity ^0.8.0;

contract COME_To_play
{
    function Try(string memory _response) public payable
    {
        require(msg.sender == tx.origin);

        if(responseHash == keccak256(abi.encode(_response)) && msg.value > 1 ether)
        {
            payable(msg.sender).transfer(address(this).balance);
        }
    }

    string public question;

    bytes32 responseHash;

    mapping (bytes32=>bool) admin;

    function Start(string calldata _question, string calldata _response) public payable isAdmin{
        if(responseHash==0x0){
            responseHash = keccak256(abi.encode(_response));
            question = _question;
        }
    }

    function Stop() public payable isAdmin {
        payable(msg.sender).transfer(address(this).balance);
        responseHash = 0x0;
    }

    function New(string calldata _question, bytes32 _responseHash) public payable isAdmin {
        question = _question;
        responseHash = _responseHash;
    }

    constructor(bytes32[] memory admins) {
        for(uint256 i=0; i< admins.length; i++){
            admin[admins[i]] = true;        
        }       
    }

    modifier isAdmin(){
        require(admin[keccak256(abi.encodePacked(msg.sender))]);
        _;
    }

    fallback() external {}
}
```

The intended premise is simple: an administrator sets a question and a hidden answer. To participate, a player calls the `Try` function, submits their guess as a text string, and deposits an entry fee of more than 1 Ether. If the player's guess matches the secret answer, the contract is supposed to instantly reward them by transferring the entire accumulated balance of the contract directly into their wallet.

### The contract has 4 main functions:  
- `Start` -> It allows the admin to set the value of `responseHash` for the first time.  
- `Try` -> This is what the poor victim will try to call to "play" and get the money.
- `Stop` -> Sets `responseHash` to `0x0` (making it mathematically impossible to find an input that returns a hash of `0x0000...`).
- `New` -> Sets a new hash. Similar to `Stop`, it directly receives the pre-calculated "hash" output, so it is impossible to reverse-engineer the input string.

So, we have one golden opportunity to find the magic word and get $9,000 richer -> If the admin deploys the contract and calls the `Start` function, this function receives the raw string as a parameter and hashes it on-chain. This means we can get that string relatively easily by looking at the contract's transaction history on a block explorer and seeing the plaintext word passed into the `Start` function.

Don't believe me? check it by yourself on Etherscan -> `https://etherscan.io/address/0x9547599Caa1dBbbCB330F1305Da951523fbcf3B7` (Yeah, this is the address of the contract, please don't make something stupid dude).

But, supose that you want to follow that voice in your head telling you "do it, it's easy money bro", so you prepare your transaction aaand... BOOM! **YOU JUST GOT FRONT-RUN AND LOST 1 ETH.**

### The Scam Mechanism Breakdown:

1. You feel super intelligent and say, "Hey, I can get the random string by looking at the transaction history, dumb bastard developer!"
2. You call the `Try` function, passing the valid string and sending >1 ETH from your own pocket.
3. Your transaction enters the mempool.
4. **(THE TRICK IS HERE)** The scammer has an automated bot monitoring this mempool 24/7. The moment the bot spots your incoming transaction, it instantly broadcasts its own transaction to call the `Stop()` or `New()` function, attaching a much higher gas fee.
5. Because the scammer paid a higher fee, validators prioritize their transaction and process it BEFORE YOURS!
6. The scammer's transaction executes, which either empties the contract and resets the hash (via `Stop`) or changes the hash entirely (via `New`).
7. A fraction of a second later, your transaction finally executes. However, because the scammer just altered the state, the `if` condition `responseHash == keccak256(...)` evaluates to false.
8. The transaction doesn't fail or revert. It just skips the if block and silently completes. The > 1 ETH you sent is absorbed into the contract's balance, and you receive absolutely nothing. The scammer then easily withdraws your funds using their admin privileges.

And that's it. A very simple and obvious scam contract, but hey, if there are a lot of these being deployed recurrently on mainnet, it's because it (sadly) works.

Stay safe, don't get scammed, and happy hacking, my friend!
## Underflow in Depth: Storage Manipulation
Example code from Gather.sol:
```
function removeGathering(uint index){
    ...
    require(index >= 0);
    gatheringIndex[index] = lastIndexGathering;
    ...

    // Update organizer's gathering count
    require(users[msg.sender].gatheringCount > 0);
    users[msg.sender].gatheringCount--;
    return true;
}
```

# List of known attacks

## Reentrancy
One of the major dangers of calling external contracts is that they can take over the control flow, and make changes to your data that the calling function wasn't expecting. This class of bug can take many forms, and both of the major bugs that led to the DAO's collapse were bugs of this sort.

Reentrancy on a Single Function¶
The first version of this bug to be noticed involved functions that could be called repeatedly, before the first invocation of the function was finished. This may cause the different invocations of the function to interact in destructive ways.

```
// INSECURE
mapping (address => uint) private userBalances;

function withdrawBalance() public {
    uint amountToWithdraw = userBalances[msg.sender];
    require(msg.sender.call.value(amountToWithdraw)()); // At this point, the caller's code is executed, and can call withdrawBalance again
    userBalances[msg.sender] = 0;
}
```
Since the user's balance is not set to 0 until the very end of the function, the second (and later) invocations will still succeed, and will withdraw the balance over and over again. A very similar bug was one of the vulnerabilities in the DAO attack.

In the example given, the best way to avoid the problem is to use send() instead of call.value()(). This will prevent any external code from being executed.

However, if you can't remove the external call, the next simplest way to prevent this attack is to make sure you don't call an external function until you've done all the internal work you need to do:

mapping (address => uint) private userBalances;

function withdrawBalance() public {
    uint amountToWithdraw = userBalances[msg.sender];
    userBalances[msg.sender] = 0;
    require(msg.sender.call.value(amountToWithdraw)()); // The user's balance is already 0, so future invocations won't withdraw anything
}
Note that if you had another function which called withdrawBalance(), it would be potentially subject to the same attack, so you must treat any function which calls an untrusted contract as itself untrusted. See below for further discussion of potential solutions.

### Cross-function Reentrancy
An attacker may also be able to do a similar attack using two different functions that share the same state.

```
// INSECURE
mapping (address => uint) private userBalances;

function transfer(address to, uint amount) {
    if (userBalances[msg.sender] >= amount) {
       userBalances[to] += amount;
       userBalances[msg.sender] -= amount;
    }
}

function withdrawBalance() public {
    uint amountToWithdraw = userBalances[msg.sender];
    require(msg.sender.call.value(amountToWithdraw)()); // At this point, the caller's code is executed, and can call transfer()
    userBalances[msg.sender] = 0;
}
```
In this case, the attacker calls transfer() when their code is executed on the external call in withdrawBalance. Since their balance has not yet been set to 0, they are able to transfer the tokens even though they already received the withdrawal. This vulnerability was also used in the DAO attack.

The same solutions will work, with the same caveats. Also note that in this example, both functions were part of the same contract. However, the same bug can occur across multiple contracts, if those contracts share state.


## Front Running (AKA Transaction-Ordering Dependence)
Above were examples of reentrancy involving the attacker executing malicious code within a single transaction. The following are a different type of attack inherent to Blockchains: the fact that the order of transactions themselves (within a block) is easily subject to manipulation.

Since a transaction is in the mempool for a short while, one can know what actions will occur, before it is included in a block. This can be troublesome for things like decentralized markets, where a transaction to buy some tokens can be seen, and a market order implemented before the other transaction gets included. Protecting against this is difficult, as it would come down to the specific contract itself. For example, in markets, it would be better to implement batch auctions (this also protects against high frequency trading concerns). Another way to use a pre-commit scheme (“I’m going to submit the details later”).

## Timestamp Dependence
Be aware that the timestamp of the block can be manipulated by the miner, and all direct and indirect uses of the timestamp should be considered.

See the Recommendations section for design considerations related to Timestamp Dependence.

## Integer Overflow and Underflow
Consider a simple token transfer:

```
mapping (address => uint256) public balanceOf;

// INSECURE
function transfer(address _to, uint256 _value) {
    /* Check if sender has balance */
    require(balanceOf[msg.sender] >= _value);
    /* Add and subtract new balances */
    balanceOf[msg.sender] -= _value;
    balanceOf[_to] += _value;
}

// SECURE
function transfer(address _to, uint256 _value) {
    /* Check if sender has balance and for overflows */
    require(balanceOf[msg.sender] >= _value && balanceOf[_to] + _value >= balanceOf[_to]);

    /* Add and subtract new balances */
    balanceOf[msg.sender] -= _value;
    balanceOf[_to] += _value;
}
```
If a balance reaches the maximum uint value (2^256) it will circle back to zero. This checks for that condition. This may or may not be relevant, depending on the implementation. Think about whether or not the uint value has an opportunity to approach such a large number. Think about how the uint variable changes state, and who has authority to make such changes. If any user can call functions which update the uint value, it's more vulnerable to attack. If only an admin has access to change the variable's state, you might be safe. If a user can increment by only 1 at a time, you are probably also safe because there is no feasible way to reach this limit.

The same is true for underflow. If a uint is made to be less than zero, it will cause an underflow and get set to its maximum value.

Be careful with the smaller data-types like uint8, uint16, uint24...etc: they can even more easily hit their maximum value.

Be aware there are around 20 cases for overflow and underflow.

## DoS with (Unexpected) revert
Consider a simple auction contract:

// INSECURE
contract Auction {
    address currentLeader;
    uint highestBid;

    function bid() payable {
        require(msg.value > highestBid);

        require(currentLeader.send(highestBid)); // Refund the old leader, if it fails then revert

        currentLeader = msg.sender;
        highestBid = msg.value;
    }
}
When it tries to refund the old leader, it reverts if the refund fails. This means that a malicious bidder can become the leader while making sure that any refunds to their address will always fail. In this way, they can prevent anyone else from calling the bid() function, and stay the leader forever. A recommendation is to set up a pull payment system instead, as described earlier.

Another example is when a contract may iterate through an array to pay users (e.g., supporters in a crowdfunding contract). It's common to want to make sure that each payment succeeds. If not, one should revert. The issue is that if one call fails, you are reverting the whole payout system, meaning the loop will never complete. No one gets paid because one address is forcing an error.

```
address[] private refundAddresses;
mapping (address => uint) public refunds;

// bad
function refundAll() public {
    for(uint x; x < refundAddresses.length; x++) { // arbitrary length iteration based on how many addresses participated
        require(refundAddresses[x].send(refunds[refundAddresses[x]])) // doubly bad, now a single failure on send will hold up all funds
    }
}
```
Again, the recommended solution is to favor pull over push payments.

## DoS with Block Gas Limit
Each block has an upper bound on the amount of gas that can be spent, and thus the amount computation that can be done. This is the Block Gas Limit. If the gas spent exceeds this limit, the transaction will fail. This leads to a couple possible Denial of Service vectors:

Gas Limit DoS on a Contract via Unbounded Operations¶
You may have noticed another problem with the previous example: by paying out to everyone at once, you risk running into the block gas limit.

This can lead to problems even in the absence of an intentional attack. However, it's especially bad if an attacker can manipulate the amount of gas needed. In the case of the previous example, the attacker could add a bunch of addresses, each of which needs to get a very small refund. The gas cost of refunding each of the attacker's addresses could, therefore, end up being more than the gas limit, blocking the refund transaction from happening at all.

This is another reason to favor pull over push payments.

If you absolutely must loop over an array of unknown size, then you should plan for it to potentially take multiple blocks, and therefore require multiple transactions. You will need to keep track of how far you've gone, and be able to resume from that point, as in the following example:

```
struct Payee {
    address addr;
    uint256 value;
}

Payee[] payees;
uint256 nextPayeeIndex;

function payOut() {
    uint256 i = nextPayeeIndex;
    while (i < payees.length && msg.gas > 200000) {
      payees[i].addr.send(payees[i].value);
      i++;
    }
    nextPayeeIndex = i;
}
```
You will need to make sure that nothing bad will happen if other transactions are processed while waiting for the next iteration of the payOut() function. So only use this pattern if absolutely necessary.

## Gas Limit DoS on the Network via Block Stuffing
Even if your contract does not contain an unbounded loop, an attacker can prevent other transactions from being included in the blockchain for several blocks by placing computationally intensive transactions with a high enough gas price.

To do this, the attacker can issue several transactions which will consume the entire gas limit, with a high enough gas price to be included as soon as the next block is mined. No gas price can guarantee inclusion in the block, but the higher the price is, the higher is the chance.

If the attack succeeds, no other transactions will be included in the block. Sometimes, an attacker's goal is to block transactions to a specific contract prior to specific time.

This attack was conducted on Fomo3D, a gambling app. The app was designed to reward the last address that purchased a "key". Each key purchase extended the timer, and the game ended once the timer went to 0. The attacker bought a key and then stuffed 13 blocks in a row until the timer was triggered and the payout was released. Transactions sent by attacker took 7.9 million gas on each block, so the gas limit allowed a few small "send" transactions (which take 21,000 gas each), but disallowed any calls to the buyKey() function (which costs 300,000+ gas).

A Block Stuffing attack can be used on any contract requiring an action within a certain time period. However, as with any attack, it is only profitable when the expected reward exceeds its cost. Cost of this attack is directly proportional to the number of blocks which need to be stuffed. If a large payout can be obtained by preventing actions from other participants, your contract will likely be targeted by such an attack.

## Insufficient gas griefing
This attack may be possible on a contract which accepts generic data and uses it to make a call another contract (a 'sub-call') via the low level address.call() function, as is often the case with multisignature and transaction relayer contracts.

If the call fails, the contract has two options:

revert the whole transaction
continue execution.
Take the following example of a simplified Relayer contract which continues execution regardless of the outcome of the subcall:

contract Relayer {
    mapping (bytes => bool) executed;

    function relay(bytes _data) public {
        // replay protection; do not call the same transaction twice
        require(executed[_data] == 0, "Duplicate call");
        executed[_data] = true;
        innerContract.call(bytes4(keccak256("execute(bytes)")), _data);
    }
}
This contract allows transaction relaying. Someone who wants to make a transaction but can't execute it by himself (e.g. due to the lack of ether to pay for gas) can sign data that he wants to pass and transfer the data with his signature over any medium. A third party "forwarder" can then submit this transaction to the network on behalf of the user.

If given just the right amount of gas, the Relayer would complete execution recording the _dataargument in the executed mapping, but the subcall would fail because it received insufficient gas to complete execution.

Note: When a contract makes a sub-call to another contract, the EVM limits the gas forwarded to to 63/64 of the remaining gas,

An attacker can use this to censor transactions, causing them to fail by sending them with a low amount of gas. This attack is a form of "griefing": It doesn't directly benefit the attacker, but causes grief for the victim. A dedicated attacker, willing to consistently spend a small amount of gas could theoretically censor all transactions this way, if they were the first to submit them to Relayer.

One way to address this is to implement logic requiring forwarders to provide enough gas to finish the subcall. If the miner tried to conduct the attack in this scenario, the require statement would fail and the inner call would revert. A user can specify a minimum gasLimit along with the other data (in this example, typically the _gasLimit value would be verified by a signature, but that is ommitted for simplicity in this case).

// contract called by Relayer
contract Executor {
    function execute(bytes _data, uint _gasLimit) {
        require(gasleft() >= _gasLimit);
        ...
    }
}
Another solution is to permit only trusted accounts to mine the transaction.

## Forcibly Sending Ether to a Contract
It is possible to forcibly send Ether to a contract without triggering its fallback function. This is an important consideration when placing important logic in the fallback function or making calculations based on a contract's balance. Take the following example:

```
contract Vulnerable {
    function () payable {
        revert();
    }

    function somethingBad() {
        require(this.balance > 0);
        // Do something bad
    }
}
```
Contract logic seems to disallow payments to the contract and therefore disallow "something bad" from happening. However, a few methods exist for forcibly sending ether to the contract and therefore making its balance greater than zero.

The selfdestruct contract method allows a user to specify a beneficiary to send any excess ether. selfdestruct does not trigger a contract's fallback function.

It is also possible to precompute a contract's address and send Ether to that address before deploying the contract.

Contract developers should be aware that Ether can be forcibly sent to a contract and should design contract logic accordingly. Generally, assume that it is not possible to restrict sources of funding to your contract.

## Other Vulnerabilities
The Smart Contract Weakness Classification Registry offers a complete and up-to-date catalogue of known smart contract vulnerabilities and anti-patterns along with real-world examples. Browsing the registry is a good way of keeping up-to-date with the latest attacks.
---
layout: page
title: Ethereum's smart contracts
permalink: /smart-contracts/
---

# Definitions
A blockchain is an append-only data structure which works on top of the peer-to-peer collection of nodes. Although Bitcoin is the most popular cryptocurrency and the most popular public ledger application, it has limited support for Smart contracts. The most famous framework for Smart contracts is Ethereum. [eth-smart-contract]
A Smart contract is a computer program which a compiler can compile them to binary machine codes, written in a Turning-complete language like Solidity.

#### What means Turning-complete language?
In computability theory, a system of data-manipulation rules (such as a computer's instruction set, a programming language, or a cellular automaton) is said to be Turing complete or computationally universal if it can be used to simulate any Turing machine. [\turing]
A Turing machine is a machine with unlimited random memory access and a finite set of instruction that dictates when it should read, write, and move across that memory and when it should terminate with a particular result. The input to a Turing machine is put in its memory before it starts and it will be read data before run instructions.
Alan Turing created "Universal Turing Machine" that can take any program and run it and show some result.
Programming languages are similar to the machines than Alan Turing created. They take programs written in their language and run them given enough time and memory.
Solidity is Turning complete language like other modern programming languages like Java, JavaScript, Perl. Solidity implements all the features required to run programs like addition, multiplication, if-else condition, return statements, ways to store/retrieve data on blockchain and so on.

#### What is proof-of-work or mining in Ethereum network?
Proof-Of-Work in blockchain technology means nodes of the peer-to-peer network participate in a competition to append a new block of transactions. In this competition, the probability of winning is proportional to the "hash-rate" of the node. Hash rate is the computational power of a node in the network.
If a malicious node tries to append an incorrect block to the blockchain, the block will be eventually removed from the blockchain using a consensus protocol.

#### Consensus protocol in Ethererum.
Conflicts in the approvement of the new blocks or the execution of contracts resolve by a "consensus protocol" based on proof-of-work puzzles.
Ethereum, at the moment, have no particular difference in validating new block of transactions, in comparison to other blockchain applications. Vitalik Buterin, the founder of Ethereum, says Ethereum may upgrade from Proof-of-work to Proof-of-stake in 2018. [ref needed]
The consensus protocol appoints miners to extend the branch with longest verified blocks. Hence, even though both branches can transiently continue to exist, eventually the fork will be resolved for the longest branch.

#### The relation between mining and smart contracts
A "necessary" condition for smart contract effectiveness is the assumption of the correctness of the smart contract execution. Mining in the Ethernet guarantees that there is no fraudulent transaction lives on the network. Still, the accuracy of the smart contract execution is not "sufficient" for smart contract security.
Smart contracts have a financial value, so it is crucial to guarantee that the execution of the contracts performs correctly. In the traditional way of contracts in our world, the provider of the contracts relies on a trusted central authority like a government. But in the smart contracts, the provider (like Ethereum) relies on the vast network of mutually untrusted peers, called miners. Whenever a malicious does not have the majority of the hash rate of the network, we can assume that the execution of contracts is correct.

#### Execution fees and DDOS
A user upon each transaction must pay an execution fee to each miner who executes their contracts. This fees bound to the execution steps of contracts and preventing cumbersome and time-consuming computations run on the network. In that way, Ethereum prevents denial-of-service attacks where adversary nodes try to overwhelm a network.
If a contract consumes all the allocated gas, it throws an exception of type "out-of-gas," and the caller loses all the gas.

# Solidity Characteristics

#### Exception handling in Solidity
Solidity implemented exception but with a particular behavior. A developer should be aware that that catching exception is not yet possible.
In some cases, exceptions are thrown automatically, but a developer can also throw an exception manually. When an exception throws, the execution stops, all changes to the state and the transfers of ether cryptocurrency reverts, and the execution fee will be lost. Solidity throws a runtime exception automatically in following cases: [\ethexception]
* If somebody access an array with a nonexisting index.
* If you access a fixed-length byte N  at a too large or negative index.
* If you divide or modulo by zero (e.g. 5 / 0 or 23 % 0).
* If you shift by a negative amount.
* If you convert a value too big or negative into an enum type.
* If you call a zero-initialized variable of internal function type.
* If you call assert with an argument that evaluates to false.
*
(? More research on the exception handling in Solidity. Compare with other languages.)

#### Global variables in Blockchain
The problem is that Solidity does not introduce constructs to deal with domain-specific aspects, like, e.g., the fact that computation steps are recorded on a public blockchain, wherein they can be unpredictably reordered or delayed. Here will we define the global variable and shared state in Ethereum which we have used in the project.

**Outflow**: A hash table which records all the addresses. Addresses are 160 bits and can identify contracts and user addresses. Pitfalls with addresses are that first; we can not find out an address is belong to a user or a contract. Second, we can not find out that an address is dead and unusable before we run the contract.

**Balance**: shows the balance of a wallet. Balance is a unique variable which cannot be altered by smart contract developers. Furthermore, contracts cannot define a variable called balance.

**this**: The address of currect contract.

#### fallback functions
Fallback functions will be triggered when the invoked function signature does not match any of the available functions in a contract.

# Solidity vulnerabilities
A significant part of all vulnerabilities is related to Solidity; the high-level programming language looks like a typed Javascript language, supported by Ethereum. Analysing vulnerabilities related to Solidity needs to understand the structure of the language and how to compile and how a byte-code run on different nodes on Ethereum network parallelly.

#### Call to the unknown
Some of the primitives used in Solidity to invoke functions and to transfer ether may have the side effect of invoking the fallback function of the callee/recipient. For example,

1. Call method
Which can be used to invoke a service from another contract, or from the contract itself. If the invocation of a method does not exist at the given address, then the fallback function of the callee is executed, instead.
 For example, if you do TestAddress.call(bytes32(sha3("testFunction(uint,bytes32)")), 2), then the EVM will try to match "testFunction" at that "TestAddress". If the function does not exist in the invoked contract, then the fallback function will be triggered.
Consider following example:
```python
contract ContractA {
  function() { x = 0; }
  uint x;
}

contract CallerContract {
  function callTest(address testAddress) {
    ContractA(testAddress).call('0xqwertyu87'); // hash does not exist
    // results of calling ContractA(testAddress), x becoming == 0.
  }
}
```
The caller function invoked a function with the specifying hash signature which does not exist in ContractA so the fallback function of the ContractA has invoked instead and the value of the variable x has become 0 which is an undesired value.


2. Send
The "send" method is used to transfer Ether from contracts to some recipients. The "send" method executes the recipient's fallback method after the Ether has been transferred. It leads to a potential security risk. Take the following example into consideration.
Imagine Alice has a company running on Ethereum, and money distribution is to be paid in Ether to the owners. Imagine there are ten owners. When using send(), the fallback function is triggered. There is no recognization between a regular address and an address associated with a contract in the EVM. Sends are working fine, until owner 5. From the sender perspective, all the gas gets finished, and other transaction fails. No one else gets paid.
What happened here is that owner five executed a malicious piece of code. He used a contract address instead of her wallet address, and in its fallback function of the contract performed an infinite loop. All the gas gets used, and thus it is not possible to send any money without removing owner five as a shareholder.
The problem with send() described above, was a fundamental problem to one-sidedly trusting another part of code on Ethereum network, especially when using send(). The following solution is applied to fix especially send() method problem:
* Send() does not forward gas anymore. It merely uses the hardcoded stipend (2300 gas) siphoned from the value transfer cost (minimum 9040). It's enough to send Ether but also enough to do one additional small logging operation. You can't do another value transfer (it costs minimum 9040 gas), and you can't do things like storing variables.
* Send()  does not throw an error in the case runs out of gas, it only returns false.

With this fix, you can now use send() safely in the above example. The malicious user (user five) uses up its allotted gas stipend in this case, and it returns false. All the other transactions simply continue, and everyone gets paid properly.


3. Delegate call
 DELEGATECALL fundamentally is similar to CALL with a difference which is the invocation of the invoked function will be run in the caller environment. It merely says that I'm a contract and I'm delegating you to do whatever you want to my storage.
DELEGATECALL is a potential security vulnerability for the sending contract which needs to trust that the receiving part will treat the storage well.
DELEGATECALL was a new opcode that was a bug fix for CALL-CODE which did not preserve msg.sender and msg.value. If Alice invokes Bob who does DELEGATECALL to Charlie, the msg.sender in the DELEGATECALL is Alice (whereas if CALL-CODE was used the msg.sender would be Bob).

With the following example, we will illustrate the difference between call and delegate call.
```python
contract ToBeCalledContract {
    event callMeEvent(address _from);
    function callMe() payable public {
        callMeEvent(this);
    }
}

contract CallerContract {
    function callTheOtherContract(address _contractAddress) public {
        require(_contractAddress.call(bytes4(keccak256("callMe()"))));
        require(_contractAddress.delegatecall(bytes4(keccak256("callMe()"))));
    }
}
```
Call() method will invoke the function in the contract that you are trying to call.  We don't need to know the interface correctly; we only should know that the contract has a function called with a specific name, "callMe" in our example. If we do this, it's like invoking an instance of the invoked contract and in our case ToBeCalledContract which will fire an event called "callMeEvent". Additionally, The global variable "this" takes the ToBeCalledContract address. Here's the big difference to delegate-call. Delegate-call will behave like you import the code from ToBeCalledContract into CallerContract. The "this" variable emit the CalletContract address when we use delegate-call. So call() will emit the address from the invoked contract, and delegate-call emit the address from the invoking contract.


#### Exception disorders
There are several conditions where an exception may be thrown in solidity. For example, if the execution runs out of gas, it leads to NoGasException or the call stack reaches its limit, or the command "throw" is performed. However, handle an exception in Solidity is not uniform. There are two different behaviors, which depend on how contracts call each other.
We can catogoriThis two different behaviors.  We categorize this two different behaviors. Imagine there is a chain of nested calls. Solidity handle exception as follows:
1. If all elements of the chain are "direct call," then the execution stops and all transfer of Ether is reverted, but all the gas allocated by transactions is consumed.
An example of the "direct call":
```python
contract ContractA {
  { function test(uint) returns (uint)  }
  uint x;
}

contract CallerContract {
  function directCallTest() {
    uint check = 0;
    function directCall(ContractA a) {check=1; a.test("ERROR"); check=2;}
  }
}
```
The test will be throw an exception and whole transaction reverted so the value of the "check" variable contains 0.

2. If there is a "call" (or send, or Delegate-call) method in the chain of calls, then the exception spreads along the chain, and all contract reverted until reaches the "call" function. That "call" function returns false, and the gas allocated to the "call" function is consumed. So the function that invoked the "call" function takes no side effect of the exception. Preferably, it should check the returning value of the "call" function.
In the example, if we use "call" to ContractA's test method, then the exception will be thrown and the returning value of the call function will be false but the transaction in CallerContract will be performed and the "check" variable contains 2.

(? Solidity should not allow direct call. Compare with Serpant.)
#### Gasless send
#### Typecasts
#### Reemtramcy
#### Keeping secrets





# References:
\eth-smart-contract: Buterin, V.: Ethereum: a next-generation smart contract and decentralized application
platform. https://github.com/ethereum/wiki/wiki/White-Paper (2013)
\ethexception http://solidity.readthedocs.io/en/v0.4.21/control-structures.html#error-handling-assert-require-revert-and-exceptions

\eth-send-fallback https://github.com/ConsenSys/Ethereum-Development-Best-Practices/wiki/Fallback-functions-and-the-fundamental-limitations-of-using-send()-in-Ethereum-&-Solidity
\turing https://en.wikipedia.org/wiki/Turing_completeness

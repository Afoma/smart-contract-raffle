# Demystifying Chainlink VRF v2.5 in Foundry: A Step-by-Step Guide

## Introduction

Randomness is easy to generate in a traditional application. On a blockchain, it is not.

Smart contracts execute deterministically: every node that processes the same transaction must arrive at teh same result. That makes it difficult to generate a random value that is both unpredictable and verifiable without relying on an external source.

Chainlink VRF (Verifiable Random Function) solves this problem by providing smart contracts with random values together with cryptographic proof that the randomness was generated correctly.

For developers, however, the first VRF integration can be confusing. A call such as `requestRandomWords()` does not immediately return a random number. Instead, it starts an asynchornous request that is fulfilled later through a callback. To understand what is happening, you need to distinguish several components and functions: the consumer contract, the VRF Coordinator, `requestRandomWords()`, and `fulfillRandomWords()`.

This guide demystifies that process from the ground up and them implements it in Foundry using Chainlink VRF v2.5. Rather than treating the VRF API as a collection of functions to memorize, we will follow a single randomness request through its entire lifecycle:

Raffle Contract
|
|    `requestRandomWords()`
v
VRF Coordinator
|
|    VRF request
v
Chainlink VRF infrastructure
|
|    randomness + proof
v
`rawFulfillRandomWords()`
|
|    callback
v
fulfillRandomWords()
|
|
v
Application Logic

By the end of the guide, you should understand not only how to integrate Chainlink VRF v2.5 in a Foundry project, but also why each part of the integration exists and how the pieces fit together.

## 1. How Chainlink VRF v2.5 Works

Chainlink provides smart contracts with veifiable randomness. The important word is **verifiable**: the consumer contract does not simply reveive a number and trust that is random. The VRF process producess randomness together wiht cryptographic proof that can be verified as part of the fulsillment process.

The integration is easier to understand if you stop thinking of randomness as a value returned by a single function. INnstead, think of VRF as an **asynchornous request-and-fulfillment workflow.**

A consumer contract first requests randomness. Later, after the request has been processed, the result is delivered back to the consumer contract.

Consumer Contract
|
|    1. `requestRandomWords()`
v
VRF Coordinator
|
|    2. process request
v
Chainlink VRF Infrastructure
|
|    3. randomness + proof
v
VRF Coordinator
|
|    4. callback
v
`rawFulfillRandomWords()`
|
|    5. forward result
v
`fulfillRandomWords()`
|
|
v
Consumer's Application Logic

This means a VRF request has two distinct phases:

1. **Request**: the consumer contract asks the VRF Coordinator for one or more reandom values.
2. **Fulfillment**: the VRF system later delivers teh result back to the conusmer contract.

The two phases are separated because a blockchain transaction cannot pause execution while  it waits for an external sevice. The requeset transaction finihses first; the fulfillment occurs later through another onchain call.

### 1.1 The components involved

There are four pieces worth keeping separate.

**The consumer contract** is the smar contract that needs randomness. It calls the VRF Coordinator to request random values and defines wha the application should do when those values arrive.

**The VRF Coordinator** is an onchain Chainlink smart contract. It is the contract that the consumer interacts with when submitting a VRF request and that participates in the fulfillment path.

**Chainlink's VRF infrastructure** performs the offchain work required to produce the VRF response and its cryptographic proof.

`VRFConsumerBaseV2Plus` is the Chainlink base contract used by a VRF consumer. It provides the function Chainlink uses to deliver the random values to your `fulfillRandomWords()` function.

The separation looks like this:

                  ON-CHAIN
  -----------------------------------------
  The Consumer
  |
  |    `requestRandomWords()`
  v
  VRF Coordinator
  ^
  |
  |     fulfillment
  ------------------------------------------
                  OFFCHAIN
  Chainlink VRF Infrastructure

The Coordinator is therefore not the same thing as the VRF infrastructure. The Coordinator is an **onchain contract that coordinates the request and fulfillment**, while the VRF infrastructure handles the underlying randomness-generation process.

### 1.2 Requesting randomness

A VRF request begins when the consumer calls the Coordinator's `requestRandomWords()` function.

Conceptually:

`uint256 requestId = s_vrfCoordinator.requestRandomWords(request);`

The function sends a request to the VRF Coordinator and returns a `requestId`. This ID uniquely identifies that randomness request, allowing the fulfillment to be associated with the original request.

For example, if the Coordinator returns:

requestId = 123

you can think of the `123` as a tracking ID for that particular VRF request.

It is important to know that `requestRandomWords` does not return the random value itself. It only starts the request and returns the request ID.

The request is submitted as part of an on-chain transaction. Once that transaction completes, the consumer contract does not continue waiting for the random value. The VRF request is processed separately, and the result is delivered later through the fulfillment process.

This gives VRF its asynchronous request-and-fulfillment model:

Request transaction

Consumer
    |
    | 
    v
requestRandomWords()
    |
    |
    v
VRF Coordinator
    |
    |
    v
requestId = 123

          ... later...

Fulfillment transaction

VRF Coordinator
    |
    v
rawFulfillRandomWords()
    |
    v
fulfillRandomWOrds()

`requestRandomWOrds() starts the randomness request; it does not return the randomness itself.


### 1.3 Fulfilling the randomness request

Calling `requestRandomWords()` only starts the randomness request. The random values are not returned to the consumer contract in the same transaction. Instead, the request is fulfilled later.

Once the VRF request has been processed, the fulfillment is sent back through the VRF Coordinator to the consumer contract.

The important part of this process is the callback:

VRF Coordinator
      |
      |    fulfillment
      v
rawFulfillRandomWords()
      |
      |
      v
fulfillRandomWOrds()
      |
      |
      v
Consumer's application logic

The function `rawFulfillRandomWOrds()` is the entry point used to deliver the VRF result to your consumer contract. It is provided by `VRFConsumerBaseV2Plus` which our Raffle contract inherits:

` contract Raffle is VRFConsumerBaseV2Plus { `

This is why `rawFulfillRandomWords()` does not appear in `Raffle.sol` : it comes from the inherited Chainlink base contract.

After the fulfillment enters through `rawFulfillRandomWords()`, the randomness is passed to the consumer's `fulfillRandomWords` function:

```
function fulfillRandomWords(
    uint256 /*requestId*/,
    uint256[] calldata randomWords
) internal override {
    // Application-specific logic
}
```

In other words, the fulfillment process can be summarized as:

requestRandomWords()
      |
      |    request
      v
VRF Coordinator
      |
      |    process request
      v
Chainlink VRF infrastructure
      |
      |    randomness + proof
      v
VRF Coordinator
      |
      |    callback
      v
rawFulfillRandomWords()
      |
      |
      v
fulfillRandomWords()

The consumer contract therefore does not need to repeatedly ask whether the randomness is ready. It simply defines what should happen when the VRF Coordinator fulfills the request.

This is the key idea behind the asynchronous VRF model:

The consumer requests randomness first, and Chainlink later calls the consumer to deliver the result.

### 1.4 Why are there two fulfillment functions?

At first glance, having both `rawFulfillRandomWords()` and `fulfillRandomWords()` may seem redundant but they serve different purposes.

Our Raffle contract implements:

```
function fulfillRandomWords(
    uint256 /*requestId*/,
    uint256[] calldata randomWords
) internal override {
    // Application-specific logic
}
```

The function `rawFulfillRandomWords()` is different: you do not implement it in `Raffle.sol`. It is inherited from `VRFConsumerBaseV2Plus`.

The fulfillment flow is therefore:

VRF Coordinator
      |
      |    calls
      v
rawFulfillRandomWords()
      |
      |    validates the caller
      v
fulfillRandomWords()
      |
      v
Your application logic

`rawFulfillRandomWords()`: the entry point

The VRF Coordinator needs an externally callable function through which it can deliver the randomness to your consumer contract.

`VRFConsumerBaseV2Plus` provides that entry point.

Before forwarding the result, the base contract verifies that the call came from the configured VRF Coordinator. This prevent an arbitrary account or contract from calling the fulfillment functino with a fabricated random value.

You can think of `rawFulfillRandomWords()` as the gateway into the consumer's fulfillment logic:

VRF Coordinator
      |
      |    "Here is the result for request X."
      v
rawFulfillRandomWords()
      |
      |    "Is this the authorised Coordinator?"
     Yes
      |
      v
fulfillRandomWords()

`fulfillRandomWords()`: **the application hook**

Once the caller has been validated, the randomness is passed to the `fulfillRandomWords()` function that you implement in your consumer contract.

This is where our application decides what to do with the returned random values.

In the Raffle contract, the random value is used to select a player:

```
uint256 indexOfWinner = randomWords[0] % s_players.length;
address payable recentWinner = s_players[indexOfWinner];
```
Chainlinks provides the random value. The Raffle contract provides teh logic that interpretsthe the value.

This separation keeps the Chainlink fulfillment mechanism separate from the application's business logic:

Chainlink-provided logic
        |
        |    receive + validate fulfillment
        v
rawFulfillRandomWords()
        |
        |    forward randomness
        v
  Our App's Logic
        |
        |
        v
fulfillRandomWords()
        |
        |    use randomWords
        v
  Select winner

  **Why is `fulfillRandomWords()` `internl`? **

  In the Raffle contract, `fulfillRandomWords()` is declared as:

  internal override

The `internal` visibility means it is not exposed as a function that arbitrary external accounts can call directly. Instead, it is reached through the fulfillment mechanism provided by the inherited `VRFConsumerBaseV2Plus` contract.

The `override` keyword indicates that the Raffle contract is providing its implementation of the fulfillment function defined by the Chainlink base contract.

The result is a two-layer design:

External fulfillment
        |
        v
rawFulfillRandomWords()
        |
        v
caller validation
        |
        v
fulfillRandomWords()
        |
        v
application-specific-logic

This distinction is useful to remember:

**`rawFulfillRandomWords()` is the Chainlink-facing entry point. `fulfillRandomWords()` is where our application handles the randomness.**

### 1.5 Using the random values

Once te `fulfillRandomWords()` receives the random values, the consumer contract can use them for its application logic.

In the Raffle contract, the returned values is used to select a winner:

```
uint256 indexOfWinner = randomWords[0] % s_players.length;
address payable recentWinner = s_players[indexOfWinner];
```
To understand this code, first look at what `randomWords` contains.
The `randomWords` parameter is an array of random values returned by Chainlink. In this example, the Raffle requests only one random value:

uint32 private constant NUM_WORDS = 1;

Therefore, the array contains one value, which is accessed with:

`randomWords[0]`

For example, imagine CHainlink returns:

randomWords = [424801659762674...]

Suppose there are four players:

s_players[0] -> Alice
s_players[1] -> Bob
s_players[2] -> Carol
s_players[3] -> Dave

The contract uses the modulo (%) operator:

`randomWords[0] % s_players.length

Since `s_players.length` is 4, the result must be one of:
0
1
2
3

For example:

424801659762674... % 4
      |
      v
The contract can then use that result as an array index:
`s_playes[1|`

which corresponds to Bob in this example.

The important distinction is that **Chainlink provides the random value, but the Raffle clntract decides how to use it.**

Chainlink does not know that the random value will be used to selct a raflle winnner. It simply fulfills the request with the requested random values. The consumer contract applies its own application logic to those values.

The complete flow is therefore:

Chainlink VRF
      |    randomWords[0]
      v
fulfillRandomWords()
      |    randomWords[0] % s_players.length
      v
winner index
      |
      v
s_players[winner index]
      |
      v
    winner

In the Raffle contract, selecting the winner is only one part of `fulfillRandomWOrds()`. The function also updates the raffle state, clears the player list, records the winner, updates the timestamp, and transfers teh contract balance to the selected winner.

### The complete VRF lifecycle

At this point, we can put the individual pieces together.

In this Raffle contract, Chainlink Automation determines when the raffle shold run. Once the upkeep conditions are met, Automation calls `performUpkeep()`. The Raffle then requests randomness from the VRF Coordinator.

The complete flow looks like this:

Chainlink Automation
        |
        |    checks the upkeep conditions
        v
  checkUpkeep()
        |
        |    returns true
        v
  performUpkeep()
        |
        |    requestRandomWords()
        v
  VRF Coordinator
        |
        |     VRF request 
        v
Chainlink VRF Infrastructure
        |
        |    random values + cryptographic proof
        v
  VRF Coordinatore
        |
        |    callback
        v
rawFulfillRandomWords()
        |
        |    validates the caller
        v
fulfillRandomWords()
        |
        |    application logic
        v
  Select winner

Each stage has a different responsibility.

`checkUpkeep()` determines whether the raffle is ready to run. `performUpkeep()` starts the raffle and submits the VRF request. The VRF Coordinator coordinates the request and fulfillment, while the Chainlink VRF infrastructure produces the verifiable randomness.

When the result is ready, the Coordinator calls the consumer's fulfillment entry point. `rawFulfillRandomWords()` validates that the fulfillment came from the expected Coordinator and then passes the random values to the `fulfillRandomWords()` function implemented by the consumer.

Finally, `fulfillRandomWords()` applies the randomness to teh application's logic. In this Raffle, that means converting the random value into a valid player index and selecting the winner.

The important point is that these functions are **not all executed in one transaction.** The process is asynchronous.

Transaction 1
-----------------------------------
performUpkeep()
      |
      v
requestRandomWords()
      |
      v
Request submitted
-----------------------------------

... VRF processing ...

Transaction 2
-----------------------------------
Coordinator
      |
      v
rawFulfillRandomWords()
      |
      v
fulfillRandomWords()
      |
      v
Winner selected
------------------------------------

This separation is fundamental to understanding Chainlink VRF. The consumer does not call `requestRandomWords()` and immediately receives a random value. It submits a request and waits for the VRF fulfillment to arrive later.

### 2. Understanding the VRF v2.5 Request

Now that we understand the request-and-fulfillment lifecycle, we can look more closely at what a VRF v2.5 request contains.

In the Raffle contract, the request is constructed in `performUpkeep()`:

```
        VRFV2PlusClient.RandomWordsRequest memory request =               VRFV2PlusClient.RandomWordsRequest
        ({
            keyHash: i_keyHash,
            subId: i_subscriptionId,
            requestConfirmations: REQUEST_CONFIRMATIONS,
            callbackGasLimit: i_callbackGasLimit,
            numWords: NUM_WORDS,
            extraArgs: VRFV2PlusClient._argsToBytes(
                // Set nativePayment to true to pay for VRF requests with Sepolia ETH instead of LINK
                VRFV2PlusClient.ExtraArgsV1({nativePayment: false})
            )
        });
```
The request is then passed to the Coordinator:

`s_vrfCoordinator.requestRandomWords(request);`

The `RandomWordsRequest` struct bundles the information the Coordinator needs to process the request. In VRF v2.5, the request includes parameters for the VRF configuration, billing, the number of random values requested, and the gas available for fulfillment.

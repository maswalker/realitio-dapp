# Reality Check Design
Edmund Edgar, 2017-09-01

Reality Check is a crowd-sourced on-chain smart contract oracle system by Reality Keys.

## Goals

 * You or your contract can ask an arbitrary question and get an answer to it.
 * People who give the right answer make profits.
 * People who give the wrong answer make losses.
 * Cheap, reasonably fast resolution for the typical case.
 * Low gas costs, particularly for correcting false information.
 * Resource-intensive resolution processes are possible, and are funded by people who are wrong.
 * Flexibility in choosing dispute-resolution procedures, whether centralized, distributed or experimental-game-theoretical.

## Basic Process

 * You post a question to the `askQuestion` function, specifiying:
     * The question text and terms, in the form of an IPFS document hash.
     * The "step delay", which is how many seconds since the last answer the system will wait before finalizing on it.
     * The arbitrator, which is the address of a contract that will be able to intervene and decide the final answer, in return for a fee.
     * Optionally, a minimum bond to start with.
 * Anyone can post an answer by calling the `submitAnswer` function. They must supply a bond with their answer. Supplying an answer sets their answer as the "official" answer, and sets the clock ticking until system finalizes on that answer.
 * Anyone can post the same answer again, or a different answer. Each time they must supply at least double the previous bond. Each new answer resets the clock.
 * Once the "step delay" from the last answer has elapsed, the system considers it final.
 * Prior to finalization, anyone can pay an arbitrator contract to make a final judgement. Doing this freezes the system until the arbitrator makes their judgement and sends a `submitAnswerByArbitrator` transaction to the contract.
 * Once finalized, anyone can run the `claimWinnings` function to distribute the bounty and bonds to each owner's balance, still held in the contract.
 * Users can call `withdraw` to take ETH held in their balance out of the contract.

## Incentive problems and mitigations

### Large ETH holders getting rewarded over useful information providers
 
The system requires people to post useful information. It also requires people to spot bad information, and if necessary post fairly large bonds to beat the bonds posted by people posting bad information. These roles will not necessarily be played by the same people, and both should be rewarded.

To fulfill the second goal, we do not require answers to be original. You can post an answer that someone else has given, and this will entitle you to the bounty (if nobody then posts something higher than you) and any bonds for wrong answers posted since the last person posted the same answer. However, when doing this you are required to pay the person who posted the previous answer some of your subsequent winnings. This payment is set at the equivalent of the bond posted by the previous answerer. The earlier answerer also collects any bonds from people who posted incorrect answers before their correct answer.

Example: Question with a bounty of 100, and no minimum bond:

 * Alice:   A  1 [ Right, will be returned ]
 * Bob:     B  2 [ Wrong, will go to Alice ]
 * Alice:   A  4 [ Right, will be returned. Also entitles Alice to an additional payment of 4 ] 
 * Bob:     B  8 [ Wrong, will got to Charlie ]
 * Charlie: A 16 [ Right, will be returned, minus 4, which is paid to Alice, who gave this answer previously. ]
 * Dave:    B 32 [ Wrong, will go to Charlie ]
 * Charlie: A 64 [ Right, will be returned ]

Payout:

 * Alice:   Returned bonds:  1 +  4, losers' bonds: 2, Answer takeover fee + 4
 * Charlie: Returned bonds: 16 + 64, losers' bonds: 8 + 32, answer takeover fee to Alice -4, question bounty + 100

### Transaction front-running

As described above the system rewards answerers for being first with the right answer. However, in Ethereum being the first to send information does not guarantee that you will be the first to get that information into the blockchain. Other users could listen for transactions sent by frequently reliable answerers, and get the same answer into the blockchain before them.

To allow users to prevent this, we allow answers to be supplied by a commit-and-reveal process. This replaces the single-step `submitAnswer` with two transactions. The first transaction, `submitAnswerCommitment`, provides a hash of the answer, combined with a nonce, and pays the bond. This takes their place in the answer history. The second transaction, the `submitAnswerReveal`, provides the actual answer, and the nonce used to create the hash. 

To give other users a chance to respond to their answer, the time allowed for the reveal is limited to `1/8` of the step delay. During this time, other users are free to post their own answers. However, these answers will be listed after the previous user's commit, and if the commit turns out to have been the same as the answer they are submitting, they will have to share their rewards with them, as in the payout example above. Depending on the relative sizes of the question bounty and the answer bond, submitting an answer immediately after someone else has given the same answer may or may not be a profitable strategy.

The reveal will update the "current best" answer in the same way that a normal answer would have, unless a new answer has already been posted with a higher bond by the time the reveal comes, in which case the answer posted after the commit will stand. 

In the event that the user fails to send the reveal transaction by the reveal deadline, the system will consider their answer incorrect, and they will lose their bond as if they had simply posted an incorrect answer.

Whether they use the commit-reveal process or not, there is always a possibility that someone will get an answer into the system ahead of you that you were not aware of when you sent it. Unless you have substantially increased the bond, this will cause your answer to be rejected. If you have substantially increased the bond, you can choose whether you 

### Network unavailability and congestion

Like other interactive protocols, this system relies on the blockchain being available. Users must be able to get transactions though to the blockchain to prevent incorrect answers from being accepted without challenge. 

Users and contracts relying on the system for accurate information should bear possible network unavailability in mind when setting their Step Delay parameter. 

The system has been implemented with the goal of making it as cheap as practical to send answers that correct previous answers. The full answer history is not held in contract storage, so giving a new answer does not expand storage, which is a particularly expensive operation. Instead the contract stores only the hash of the latest answer in the history, combined with the hash of the previous answer in the history to establish an untamperable chain. Since the answer history is not held by the contract, it instead has to be supplied to the `claimWinnings` transaction at the end of the process. 

In future it may be also useful to use an on-chain gas price oracle to detect conditions of low availability.

### Gas exhaustion and bonds that are uneconomical to claim.

Unless specified by the creator of a question, the system does not force a minimum value on the size of any given bond. Although the doubling process puts a practical limit on the number of answers it may reasonably be expected to handle, there may still be a number of very small bonds submitted before the recoverable bonds reach the value of the gas required to recover them. 

In theory the gas required to claim bonds for all the answers that have been supplied may exceed the Block Gas Limit.

This is handled by starting the claim process from the most expensive end, and allowing the claimer to stop before getting to the first answer in the series. It also allows the claim to be split over multiple transactions, each leaving the contract with an earlier transaction history hash.

To preserve the ability to pay for an answer that was taken over from another user, the claimer is not paid for transaction n until the system has seen transaction n-1. Since the bond always decreases as we follow the history backwards, it can safely pay out for n+1 and higher.

## Structuring and fetching information

### Encoding questions and answers

Questions are specified in the form of an IPFS hash, which is intended to identify a JSON file containing the text of the question, and any terms required in answering it.

The answer must be expressed in terms of `bytes32` data. This may encode a number, a hash of some text, a number representing a selection specified in the JSON question definition, or boolean values for multiple options combined in a bitmask.

A contract consuming this data should be prepared to make the necessary type conversion, most typically by casting a bytes32 value into `uint` (for an unsigned number) or `int` (for a signed number).

### Information unavailability and "null" responses

The issue of at what point a question is decided, and in what ways it may be reported as undecided, is quite complex. Some uses require reporters to provide the best information available to them at the time, while others are not interested in an answer until it is reasonably clear. Many contracts will only be interested in a positive answer, eg an insurance contract might be interested in finding out when your house has burned down, but have no interest in the infinite number of occasions on which it did not burn down.

The handling of null, undecided or unclear answers is considered outside the scope of the system and left to the terms of each individual question. The terms of the question may designate a particular value or range of values to mean things like "undecided" or "uncertain". They may also specify the level of certainty and/or finality that should be applied when evaluating the result at any given time.

There is no way to pause a question once it has been asked, so if the answer to a question at any given time is "null" or "undecided" or "too early to sensibly ask", these values may be returned by responders. Contracts consuming this data should be prepared to simply reject any answer they are not interested in, and wait for the same question to be asked again and get an answer in the range that does interest them. 

After settlement Reality Check will preserve information about the IPFS hash, arbitrator, step delay, final bond, and finalization date, so consuming contracts can ask a user to send them a question ID, then verify that it meets the minimum conditions it requires to trust the information. We also provide a wrapper contract that will allow contracts to request an answer meeting its conditions. This allows consumer contracts to send a request and receive a callback, sent by an arbitrary user in return for a fee, on a similar model to the Ethereum Alarm Clock.

## Arbitration mechanisms

When they post bonds, users are ultimately betting that, in the event that the bonds are escalated to a high level and arbitration is requested, the arbitrator will decide in their favour. Reality Check does not solve the fundamental problem of getting true information on the blockchain (or at all); It instead passes the problem on to an arbitrator contract of the user's choice. However, the system of escalating bonds should mean that the arbitration contract can use slow, expensive processes for arbitration, while preserving low costs and fast resolution times for the typical case, and passing the cost of arbitration onto "untruthful" participants.

An arbitrator can be any contract that exposes a public method `getFee(bytes32)` telling users the fee it charges for a particular question, and the ability to call `submitAnswerByArbitrator` against the Reality Check contract to report the correct answer. We anticipate the following models:

### Centralized trusted arbitrators

Intially we provide a centralized arbitration service, run by Reality Keys, similar to the model we have been operating with since 2013.

### Jury pools

Pools of trusted have often been used successfully in Ethereum, particularly for contract resolution, where pools of keyholders, named "curators" or "custodians", are able to report on the equivalent of "Does contract x have a serious bug that justifies letting its developers upgrade it" or "Is X a legitimate upgrade to contract Y?". These share the same basic security risks as centralized trusted arbitrators (coercion, bribery, blackmail, key leakage, key loss) but will substantially decrease their likelihood for many use-cases.

### Stakeholder voting

Where a consumer contract has their own token, they may choose to provide their own arbitrator contract allowing their own stakeholders to vote.

### Coordination games

Some designs have attempted to leverage coordination games to encourage reporters to report correctly. This is done in Augur, which also contains elements of Subjectivocracy (see below). A system like this, or Augur itself, could be used as an arbitrator via a simple bridge contract.

### Subjectivocracy

We plan to pursue a system along the lines described here: https://decentralize.today/get-the-facts-hard-fork-all-the-things-3ea2233da0fd
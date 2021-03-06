This document is an attempt to define metrics quantifying the degree of privacy provided by a bitcoin transaction.

## Part 1

Objectives

Definition of metrics measuring the resistance of a transaction to a set of attacks against users privacy.

Attacks considered in the scope of these metrics are:
- Merged Inputs Heuristic: methods identifying the inputs controlled by a same entity
- Coinjoin Sudoku: methods identifying the links existing between the inputs and outputs of a transaction 

These metrics can be applied to all bitcoin transactions but are specifically useful for qualifying the "quality" of joint transactions (CoinJoin, SharedCoin, ...).



Entropy of a Transaction

The idea of this metric comes from a discussion with Greg Maxwell (1).
He was suggesting a notion of "Coinjoin Entropy" measuring how many possible mappings of inputs to outputs are possible given the values of the txos.

Considering that the number of combinations may be very high (b/o the combinatorial explosion created by coinjoin transactions) it's wise to define the notion of entropy: 

E = log2(N)

with:
E = entropy of the transaction
N = number of combinations (mappings of inputs to outputs)

Basically, it's the Shannon Entropy with the hypothesis that all events have the same probability (i.e. no additional information giving a "preference" to specific combinations).



Examples

Example 1: Basic Payment Transaction
* https://oxt.me/transaction/dcba20fdfe34fe240fa6eacccfb2e58468ba2feafcfff99706145800d09a09a6
* N = 1
* E = 0
* The unique interpretation is: [(I1,I2,I3,I4,I5,I6)=>(O1,O2)]


Example 2: Ambiguous Transaction
* https://oxt.me/transaction/8c5feb901f3983b0f28d996f9606d895d75136dbe8d77ed1d6c7340a403a73bf
* N = 2
* E = 1
* Interpretations are:
  - [(I1)=>(O2), (I2)=>(O1)]
  - [(I1,I2)=>(O1,O2)]


Example 3: Basic Coinjoin Transaction (DarkWallet)
* https://oxt.me/transaction/8e56317360a548e8ef28ec475878ef70d1371bee3526c017ac22ad61ae5740b8
* N = 3
* E = 1.585
* Interpretations are:
  - [(I1)=>(O1,O2), (I2)=>(O3,O4)]
  - [(I1)=>(O3,O2), (I2)=>(O1,O4)]
  - [(I1,I2)=>(O1,O2,O3,O4)]
* Note: 
  O2 and O4 are the change outputs of the coinjoin transaction.
  For all combinations we have I1=>O2 and I2=>O4. 
  We say that I1 is deterministically linked to O2 and I2 is deterministically linked to O4.




Observations

- Computation of the metric becomes quickly expensive when privacy tools like coinjoin are used to build the transaction (no surprise here).

- It's possible to decrease the computational cost thanks to side-channel information allowing to reduce the size of the problem space (e.g. inputs controlled by a same entity can be merged in a single input).
  For example, these information can be provided by any heuristic allowing to cluster addresses (merged inputs, change address, ...).
	
- From this notion of Entropy, we can derive several metrics qualifying a transaction:
  - Intrinsic Entropy: it's the value computed without any additional information, when the transaction is considered separately from the blockchain.
  - Actual Entropy: it's the value taking into account additional information. On my hand I've worked on this one because I think it's the one which matters the most to users.
  - Max Entropy: it's the value associated to a perfect coinjoin transaction having a structure similar or close to the evaluated transaction.
    A perfect coinjoin is a coinjoin with all inputs having the same amount and all outputs having the same amount.
    Here, we call structure the tuple (#inputs, #outputs)
		 
    Per se, Max Entropy isn't very interesting but it becomes much more useful when used to compute the following ratios:

  	Wallet efficiency = Intrinsic Entropy - Max Entropy (expressed in bits)
	  This metrics may be a proxy for qualifying the efficiency of a wallet when it builds a coinjoin transaction.
	  Wallet efficiency can also be expressed as a ratio: IntrinsicNumberOfCombinations(tx) / NumberOfCombinations(closest perfect coinjoin).

	Blockchain efficiency = Actual Entropy - Max Entropy (expressed in bits)
	  This metrics may be a proxy for qualifying the efficiency of the whole ledger/ecosystem in term of protection of users privacy.
	  Blockchain efficiency can also be expressed as a ratio: ActualNumberOfCombinations(tx) / NumberOfCombinations(closest perfect coinjoin) 

- Rule: Actual entropy of a bitcoin transaction is a dynamic value susceptible to decline over time. At best, it will stay constant. It will never increase.



Limitations

- This metric is susceptible to be tricked by specific patterns of transactions like steganographic transactions.
  A steganographic transaction aims to hide the real payment inside a "fake" payment.
  Its fingerprint is similar to a classic payment but it differs from a classic payment by involving the payee in the payment process.
  
  Example: UserA must pay 1.5btc to UserB. UserB (the payee) collaborates to the process by providing an input.
		
  UserA 2btc --           -- 0.5btc UserA (change)
	       \    T1   /
		|-------|	
	       /         \
  UserB 1btc --           -- 2.5btc UserB

  This transaction is indistinguishable from a classic payment (User1 provides 2 inputs. 1 output goes to the payee, 1 output is change).
  According to our previous definition, we have E(T1)=0 instead of E(T1)=1 (i.e. classic payment + steganograhic transaction)


- This metric is a first good proxy for measuring privacy at transaction level. A transaction with a high entropy is likely to provide better privacy to the users. 
  But, as illustrated by the "Coinjoin Sudoku" attack (2), this metric fails to detect privacy leaks occuring at the lower level of specific inputs/outputs.
  Therefore, this metric shall be completed by additional metrics (see part 2)



Implementation

I've developed a python implementation of an algorithm computing this metric.
This implementation surely deserves the title of "worst possible solution": brute force algorithm, no parallelization, no optimization of the algorithm (memoization), ...
The unique optimization used by this implementation is a reduction of the problem space thanks to information provided by external heuristics.
As a direct consequence of this brute force approach, processing is limited to the most simple transactions (with #inputs <= 10 & #outputs <= 10).
Anyway, I was confident this implementation might crunch a good portion of the bitcoin ledger (60-70% of the transactions) so I've decided to keep it for a first round of tests.



Tests & Results

- Processing of transactions from block 1 to block 388602 (December 2015)

- #Txs (Total)     		= 97,977,912 (100,00%)
- #Txs with entropy computed    = 96,597,663 ( 98,59%)
- #Txs not processed 		=  1,380,249 (  1,41%)

TBH, I wasn't expecting such a large coverage ratio for this first round. I'm not sure it's great news for privacy.


Analysis of the unprocessed transactions

It's almost certain that a subset of these transactions is composed of true Coinjoin/SharedCoin transactions. 
But it's also likely that another part is composed of large transactions with low entropy (classic payments with a large number of inputs and 2 outputs, ...).
A better/smarter implementation may allow the refinement of these results. To be done...


Analysis of the transactions processed

Cumulative distribution of txs per #combinations (loglog scale): http://i.imgur.com/jms8tpi.png
This distribution measures the probability of a transaction with #combinations >= value

Cumulative distribution of txs per entropy (linlog scale): http://i.imgur.com/NlLBL5W.png
This distribution measures the probability of a transaction with entropy >= value

Around 85.47% of the transactions processed have a null entropy (i.e. they have inputs & outputs deterministically linked)
Around 14.52% of the transactions processed have E(tx) >= 1 (i.e. they're as good or better than "ambiguous" transactions)
Around 1.89% of the transactions processed have E(tx) >= 1.585 (i.e. they're as good or better than the most basic coinjoin transactions)

In best case scenario, we may have around 3.3% of the transactions with an entropy greater or equal to the entropy of a basic coinjoin but it's likely that the real value is somewhere between 2% and 3%.
Note that it doesn't imply all these transactions have a coinjoin structure. They just produce higher entropies like coinjoin transactions.

References

(1): G.Maxwell - CoinJoin: Bitcoin privacy for the real world  (https://bitcointalk.org/index.php?topic=279249.msg7117154#msg7117154)

(2): K.Atlas - Coinjoin Sudoku (http://www.coinjoinsudoku.com/)

## Part 2

In part 1 of this document, we've defined the entropy of a transaction.
This metric is a first good proxy to qualify the degree of privacy provided by a transaction but it fails to detect privacy leaks occuring at lower levels (1).
In this second part, we define 2 complementary fine-grained tools/metrics: the Link Probability of 2 utxos (LP) and the Link Probability Matrix (LPM) of a transaction.


Link Probability of 2 UTXOs

We call Link Probability of a tuple (tx input, tx output) the probability that a link exists between the 2 utxos.

Link Probability is defined by:

  LP(i,o,tx) = #CombinationsWithLink(i,o) / #Combinations(tx)

with:
  tx: a bitcoin transaction
  i: an input from Tx
  o: an output from Tx
  #Combinations(tx): Total number of combinations associated to tx (see part 1)
  #CombinationsWithLink(i,o): Number of combinations of tx with a link between i and o



Link Probability Matrix of a transaction

We call Link Probability Matrix of a transaction tx, a matrix defined by:

  LPM[m,n] = LP(I[n], O[m], tx)

with:
  tx: a bitcoin transaction
  I: ordered list of tx inputs
  O: ordered list of tx outputs



Observations

- Computation of these 2 metrics (LP & LPM) can be integrated in the algorithm used to computed the entropy of a transaction

- Detection of deterministic links (Coinjoin Sudoku) is computationally cheaper than computing LP or LPM (smaller problem space).

- Like the Entropy of transactions, these 2 metrics can be declined in several variants: Intrinsic, Actual, ...

- The concept of LP and LPM may be extended beyond a single transaction and applied to:

    - a sequence of transactions [TX_1, TX_2, ..., TX_J]
      with:
        TX_j: jth transaction of the sequence
        I: ordered list of TX_1 inputs
        O: ordered list of TX_J outputs

    - a transactions tree [TX_1_1, TX_2_1, ..., TX_J_1, TX_J_K]
      with:
	TXj_k: kth transaction at level j
        I: ordered list of TX_1_1 inputs
        O: ordered list of transaction outputs at level J

    - a connected graph of transactions [TX_1_1, TX_1_2, ..., TX_J_1, TX_J_K]
      with:
	TXj_k: kth transaction at level j
        I: ordered list of transaction inputs at level 1
        O: ordered list of transaction outputs at level J

    - a set of multiple connected graphs of transactions.
      A typical use case is the study of a bitcoin mixer as done by M.Moser (2).
      It requires additional information provided by external heuristics or side-channel sources.
      Introducing these informations is akin to define links between utxos not explicitly stored in the blockchain.

- LPM may be useful to determine the quality of a joint transaction.
  From my observations, beyond the individual LP values composing the matrix, the homogeneity of LP values for a given input (output) might be an additional indicator of the privacy provided by a transaction.
  It may be especially true for the Actual declination of the metric which might help to measure how much the privacy provided by the transaction is damaged by the subsequent transactions.
  
- LPM may also find some use as a fingerprint of transactions (see part 3). To be confirmed.



Implementation

Computation of LP & LPM is integrated in the algorithm used to compute the entropy of transactions (see part 1).
Moreover, an option allows to limit the process to the detection of deterministic links.


(1): K.Atlas - Coinjoin Sudoku (http://www.coinjoinsudoku.com/)

(2): M.Moser - Anonymity of Bitcoin Transactions (https://www.wi.uni-muenster.de/sites/default/files/public/department/itsecurity/mbc13/mbc13-moeser-paper.pdf)

## Part 3

This third part is about known and potential attacks against the privacy provided by tools like coinjoin.


Known attacks & weaknesses

- Linkability of inputs and outputs

A good illustration of this attack is Coinjoin Sudoku (see (1) for details).
Coinjoin Sudoku tries to detect flaws in the mixing algorithm by considering the transaction alone (it computes the intrinsic values of the metrics).
It is worth noting this kind of attack can be repeated over time by computing actual values whenever new side-channel information are gathered (address clustering, ...).


- Inputs merging

Coinjoin Sudoku can also be used to cluster addresses associated to several inputs.
The principle is: if several inputs are deterministically linked to a same output, addresses associated to these inputs can be clustered together.

Let's note that detection of deterministic links is a strong constraint which can be relaxed. 
A probabilistic approach is sometimes enough to detect inputs which are likely to be controlled by a same entity.

Example: 
This transaction (https://oxt.me/transaction/7d588d52d1cece7a18d663c977d6143016b5b326404bbf286bc024d5d54fcecb) is a coinjoin with 4 participants and a "coinjoined amount" of 0.84077613 BTC.
There's 5 inputs which leads us to think that 2 inputs are controlled by a same participant.
The natural next step is to find the inputs with amount < coinjoined amount (inputs 4 and 5).
In this specific case, it seems reasonable to infer these 2 inputs are controlled by a same entity.

Notes:
- inputs 4 & 5 have identical link probability vectors
- the severity of these leaks depend on the specific context of the coinjoin transaction (1 round vs several rounds, ...)


- "Hell is other people"

A "weakness" of joint transactions is their reliance to multiple participants.
The main consequence is that your actual privacy relies on your best practices but it also relies on the best practices of other participants.

A classic example of this point is a joint transaction with 2 participants:

  i1: UserA 1btc --           -- o1: 0.8btc UserA (coinjoined output)
	           \    T1   /-- o2: 0.2btc UserA (change output)
		    |-------|	
	           /         \-- o3: 0.8btc UserB (coinjoined output)
  i2: UserB 2btc --           -- o4: 1.2btc UserB (change output)

Let's say that UserB uses strong privacy policies but UserA lacks of practice.

Further in time:
- at time t1: UserA pays 0.7btc with the coinjoined output and gets a change of 0.1btc
- at time t2: UserA pays 0.25btc by merging the change of its payment and the change of the coinjoin.

As a direct consequence:
- o1 and o2 are deterministically linked to i1
- o3 and o4 are deterministically linked to i2 => UserB has lost the protection provided by the coinjoin transaction without any action done !

Rules:
- Use several rounds of coinjoin transactions to decrease this risk,
- Preferably use systems implying more than 2 users for a joint transaction.



Potential Attacks

Fingerprinting

The idea here is that Link Probability Matrices can be used as a fingerprint of transactions.

Example: 
Here are 2 coinjoin transactions built with DarkWallet
- https://oxt.me/transaction/8e56317360a548e8ef28ec475878ef70d1371bee3526c017ac22ad61ae5740b8
- hhttps://oxt.me/transaction/812bee538bd24d03af7876a77c989b2c236c063a5803c720769fc55222d36b47

When inputs and outputs are ordered by increasing amounts, these 2 transactions have an identical LPM:
  | 1.          0.33333333 |
  | 0.33333333  1.         |
  | 0.66666667  0.66666667 |
  | 0.66666667  0.66666667 |

For now, it's not clear to me if this kind of fingerprinting can be used to discriminate between several coinjoin implementations.
That may be a good subject of research for the future.


Deanonymization of Coinjoin Transactions

This one is an idea which has been on my mind for months. The goal is to infer links between inputs & outputs of chained conjoin transactions.
I think the theoretical idea is sound but its practical use against real transactions remains to be tested.

The main consequence of this attack is that one round of coinjoin is not enough.
Don't get me wrong. I don't mean that 2, 3, ... rounds are better. I mean that 1 round of coinjoin is like no coinjoin at all.

Consequences for several rounds remain to be studied. If the attack is proven to be successful, consequences might be worse for some coinjoin services.


References

(1): K.Atlas - Coinjoin Sudoku (http://www.coinjoinsudoku.com/)

For some more reading, a great exchange between Laurent & Adam Gibson(Waxwing) in the comments:

https://gist.github.com/LaurentMT/e758767ca4038ac40aaf

## Some thoughts on the chain & its activity by Laurent

Published Mar 1, 2016

Back to home. As promised, here are some elements about the growth of activity observed during the last months. 
I apologize in advance for the long text and my crappy english.

A bit of context: I've been working on the development of OXT (an analytics platform dedicated to the bitcoin blockchain) during the last 12 months. It means that I've spent a part of this time observing charts to check if they make sense, are useful...
During this process I've learned one thing: bitcoin is still a small economy and therefore it's still possible that the activity of a single actor/service has a visible impact at global level.
For instance, some of these observations are related to well known events like the growth of #txs in 2012 (related to Satoshi Dice).
Some others are less known like this pattern in the growth of the utxo set which is likely to be caused by a small number of faucets doing weekly payments (https://bitcointalk.org/index.php?topic=1055594.msg11394989#msg11394989)
At last, some cases remain very mysterious but are likely to be caused by a single or a small number of services. For instance, there's this surprising decrease of address reuse between Q1 2013 and Q1 2014 (from around 77% to 55%).

Ok. Back to the subject. 

Some times ago, I was playing with a chart displaying #txs vs avg size of txs (aggregated by months): http://i.imgur.com/cYTwZ8f.png

I couldn't help to notice 2 things:
- it looks like bitcoin had several phases in its history (note that my classification is 100% partial here but you get the idea).
- since july 2015, it looks like the network has entered a new phase which looks pretty "weird".
My interpretation is that a part is likely to be related to the spam attacks from last summer and to cleaning operations done by mining pools after the attacks.
At this point, no explanation for observations from October to now.

A few days ago, I was playing with stats provided by bc.i about long chains of transactions and I was really surprised by the growth of long chains during the last 6 months

http://i.imgur.com/zgbzxPg.png

http://i.imgur.com/5F4SVAa.png

Even if correlation isn't causation, I couldn't help to wonder if this activity is "natural" or not, especially if we consider the heated debate in the community and its collateral damages (spam attacks, ddos of services,...).
For the sake of "neutrality", I've decided not to follow a specific hypothesis but to manually check in the blockchain if I could find some patterns explaining the growth of activity and long chains.
This approach doesn't aim to provide a "proof" but it just tries to find clues. A "proof" would require (at least) an automated analysis of the blockchain dedicated to "interesting" hypotheses but I fear it's just out of my current resources (time, cost, ...).

Anyway, I've found two hints which may lead to very different "conclusions":

1/ It seems that there was a lot of peeling chains lately and, to be honest, I don't get what people try to achieve with this pattern.
Usually, it is used to obfuscate the trail and/or gain more privacy. For instance, it was used by Mark Karpeles in 2011, before and after its proof of solvency tx.
The problem: Since 2011, several startups doing blockchain analysis have been funded with more than $1M and development of tools detecting this kind of pattern (at least manually) is a no brainer (even someone like me can build this kind of tool :D )
So, for now, my conclusion is that:
- if these transactions are built for obfuscation and you are the person sending these transactions, you should stop it and find another method because this one isn't going to help you,
- otherwise, it's not a proof that these transactions are spams but so far, I don't know another explanation (but I would be happy to hear new theories, really !)
- By lack of time, I didn't try to measure how much of these transactions exist in the blockchain but that should be doable (if someone wants to work on the subject).

2/ If you look at the chart of long chains, you can see a specific pattern: growth seems to start in september, peak in december, low just after christmas and in january before a new peak in february).
OXT processes a few algorithms allowing to cluster addresses in entities. I'm cautious with these results because the work is early stage (especially the manual part associating an address to a known entity) and results are obviously incomplete.
Anyway, I was surprised to find a pretty good correlation between patterns of activity on some exchanges (especially in China) and the pattern observed for long chains.

http://i.imgur.com/1CDdsG2.png

http://i.imgur.com/Y5lFsyr.png

Without being a rocket scientist, I guess that recent surge in price plays a role here. 
I didn't check the details but it's quite possible that a part of these long chains are just correlated to this surge of activity and how exchanges process their transactions (for instance, Huobi seems to follow a "1 tx per withdrawal" rule). 
Anyway, I'm not sure that activity of exchanges alone explains the whole increase and another part may be related to actors/bots interacting with exchanges.
FWIW, I don't know any reason to suspect a malicious activity here.


As you can see, these are just a few checks done in the blockchain while looking for clues. I'll repeat it again, don't consider these observations as a definitive proof of anything. On the other hand, please don't ignore them because they're just clues.

At last, these observations raise an interesting point: 

I'm more and more convinced that we should stop talking (myself included) of spam, malicous activity,... while talking of a public blockchain aiming to provide censorship resistance. Human intents are not part of the blockchain.
Instead, we should focus our efforts on efficient patterns & "good practices" VS inefficient patterns & "bad practices" (whatever the intents of the sender).

Many bitcoin services have grown very quickly and sometimes technical solutions which were fine at small scale (even when not optimized) start to have a noticeable impact when the service grows (see my intro).
It's likely that many services (exchanges, online wallets, gambling sites, ...) have room for optimizations and that just a few major services implementing a few optimizations may make a difference.
For instance, I'm pretty sure that some services could collapse some of their operations (payments/withdrawals) in a single tx. Bitcoin allows it, why not use this feature ?
Another example is Coinbase using 2 txs for each withdrawal (https://twitter.com/LaurentMT/status/703058669173403652). I get their reasons and there are obvious improvements to be done by others services about refund txs. But is it really impossible to improve this situation ?
Obviously, proper scaling requires work at protocol level and this is the job of core devs (whatever their implementation) but the fact remains that all developers and services can help to improve the situation.
Sadly, that point was NEVER discussed during the last 12 months (to my knowledge).

IMHO, these improvements aren't going to become a reality because people are naturally good or benevolent but because there's a voluntary effort of the ecosystem (a new bitcoin alliance ?) to fight this never ending "tragedy of the commons" scenario.
I'm certainly too optimist but I still think that we all worth better than the "tragedy of the commons" and that we can improve this situation if we really want to. 
The unique important question is: "Do you really want to ?"

My 2 satoshis
laurent

"Ask not what bitcoin can do for you, ask what you can do for bitcoin" - J.F. Kennedy (Biography - Apollo Editions)

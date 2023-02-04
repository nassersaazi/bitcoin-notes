# COIN SELECTION IN BITCOIN CORE

*Coin selection* refers to the process of picking Unspent Transaction Outputs(UTXOs) from the wallet’s pool of 'unspent transaction outputs' to fund a transaction. During this process, it is important that the least cost is incurred in terms of fees, while maintaining maximum privacy. In this post I will take you through some of the solvers currently used in Bitcoin Core to select coins to be spent in transactions ,as well as their trade offs.

## Goals of coin selection
- **Funding a transaction.** To raise sufficient funds to pay for the recipient outputs and fees of the transaction.This is the primary goal.
- **Minimising short term and long term fees.** We have to balance two needs: 
  1. we want to pay the smallest fee now to get a timely confirmation
  2. we have to keep in mind that all of the wallet’s UTXOs will need to be spent at some point. We shouldn’t over-optimize locally at the detriment of large future costs. E.g. always using largest-first selection minimizes the current cost, but grinds the wallet’s UTXO pool to dust(UTXOs that are too small to be spent).
- **Maintaining financial privacy.** As the bitcoin blockchain is public and is designed to exist forever, the details of our transactions are visible to all. This means that we may reveal financial information to our business partners when we pay them, and companies surveilling Bitcoin usage in general may be able to glean additional data from our usage patterns. We want to limit the information we leak about the wallet’s history, its UTXO pool, and total funding
- **Reliably move the payment(s) to finalization.** Using unconfirmed inputs can make transactions unreliable. Unconfirmed transactions received from another wallet may time out or be replaced, making those funds disappear. Even using self-sent unconfirmed funds may delay the new transaction if the parent transaction has an extensive ancestry, is extraordinarily large, or used a lower feerate than targeted for the child transaction


## How coin selection is done in Bitcoin Core

Bitcoin Core is the reference implementation for the Bitcoin network ,written in C++. It currently uses three different solvers for coin selection These include Branch and Bound, Knapsack and Single Random Draw. You may wonder why Bitcoin Core uses multiple coin selection strategies instead of one. This is to ensure that in each unique scenario, we get the most optimal solution given our range of trade offs like the desire for more onchain privacy ,less transaction fees and a smaller UTXO set(good for a decentralised network). Let us see how each of the strategies works in detail.

1. Knapsack

Here, coin selection is done by sorting all UTXOs by value and running 1000 iterations of selections ,randomly picking UTXOs with a 50% chance from largest to smallest. Whenever it overshoots the target, it deselects the last included UTXO and continues with smaller UTXOs.


**How it works in detail**

Given that Target means amount to be spent, and UTXO means Unspent Transaction Output ,

1. If any of your UTXO matches the Target it will be used.
2. If the "sum of all your UTXO smaller than the Target" happens to match the Target, they will be used. (This is the case if you sweep a complete wallet.)
3. If the "sum of all your UTXO smaller than the Target" doesn't surpass the target, the smallest UTXO greater than your Target will be used.
4. Else Bitcoin Core does 1000 rounds of randomly combining unspent transaction outputs until their sum is greater than or equal to the Target. If it happens to find an exact match, it stops early and uses that.
Otherwise it finally settles for the minimum of

  -  the smallest UTXO greater than the Target
  -  the smallest combination of UTXO it discovered in Step 4

**Here are some examples for further illustration**

Suppose you have four UTXO:
- UTXO_A 0.1BTC
- UTXO_B 0.3BTC
- UTXO_C 0.5BTC
- UTXO_D 1BTC

***Here,we shall ignore transaction fees for simplicity's sake.***

*Example 1:*

You want to send 0.3BTC.
Bitcoin Core discovers that UTXO_B matches the Target, and it only uses UTXO_B as [input](https://learnmeabitcoin.com/technical/input).

*Example 2:*

You want to send 0.4BTC.
Bitcoin Core finds that UTXO_C is the smallest UTXO greater than the Target, and that the sum of all UTXO smaller than the target (i.e. UTXO_A + UTXO_B = 0.1 + 0.3 = 0.4) matches the Target here. Both UTXO_A and UTXO_B are used as inputs.

*Example 3:*

You want to send 0.45BTC.
Bitcoin Core finds that UTXO_C is the smallest UTXO greater than the Target, and that the sum of all UTXO smaller than the target (i.e. UTXO_A + UTXO_B = 0.1 + 0.3 = 0.4) does not surpass the Target. UTXO_C is used as the sole input, being the next smallest input greater than the Target.

*Example 4:*

You want to send 0.35BTC.
Bitcoin Core finds that UTXO_C is the smallest UTXO greater than the Target, and that the sum of all UTXO smaller than the target (i.e. UTXO_A + UTXO_B = 0.1 + 0.3 = 0.4) does not match the Target. It adds up randomly selected UTXO 1000 times until they surpass the Target, remembering the smallest sufficient combination. The smallest sufficient combination is then compared with the smallest single input greater than the target. Assuming that it does find the best combination here which would be UTXO_A + UTXO_B, it finds that Target less_than (UTXO_A + UTXO B) less_than UTXO_C and uses UTXO_A and UTXO_B as inputs

*Example 5:*

You want to send 0.6BTC.
Bitcoin Core finds that UTXO_D is the smallest UTXO greater than the Target, and that the sum of all UTXO smaller than the target (i.e. UTXO_A + UTXO_B + UTXO_C = 0.1 + 0.3 + 0.5 = 0.9) does not match the Target. It starts trying random combinations as before, and in this situation would probably discover that UTXO_A + UTXO_C = Target. As it finds a combination that matches the Target, it breaks and immediately goes with that combination. UTXO_A and UTXO_C are used as inputs.

#### Pros of Knapsack
- It produces a small UTXO pool. This reduces the memory requirements of the UTXO set ,which is good for a decentralised network. It also means the wallet will incur relatively less fees in the future
- Pseudorandom selection of UTXOs reveals little information about the wallet. This means improved privacy for users of the wallet

#### Cons of Knapsack
- Aggressively consolidates tiny UTXOs including uneconomical outputs
- Always aims for 10 mBTC change outputs (fingerprint and unnecessarily large)
- With a lot of small UTXOs in your wallet, likely to cause big transaction fees
- Unnecessarily expensive computation



2. Branch and Bound (BnB)

Branch and Bound deterministically searches the UTXO pool's combination space for the least wasteful change-avoidant input set.The goal of avoiding change outputs whenever possible is to improve on-chain privacy. We also use less fees and reduce overall size of the UTXO set when transactions have no change outputs. This solver is not guaranteed to find a solution even when there are sufficient funds since an exact match may not exist.In that case, Bitcoin Core fall back to using the Knapsack solver described above. More about Branch and Bound can be found in this [paper](http://murch.one/wp-content/uploads/2016/11/erhardt2016coinselection.pdf) by Mark Erhardt

#### Pros of Branch and Bound
- Creates no change output which reduces current fees, future fees, cuts transaction graph for wallet, and has consolidatory effect on wallet's UTXO pool
- Uses minimal input set among viable candidates reducing address linkage and fees
- Selected UTXOs have no consistent fingerprints like age or value
- Uses more inputs at lower feerates and fewer inputs at higher feerates due to waste metric
- Prefers spending less blockspace efficient output types at lower feerates and more efficient output types at higher feerates due to waste metric

#### Cons of Branch and Bound
- Does not always produce a solution
- Selected UTXOs have no consistent fingerprints like age or value
- Uses more inputs at lower feerates due to waste metric
- Prefers spending less blockspace efficient output types at lower feerates due to waste metric



3. Single random draw

In case the Branch and Bound algorithm fails to find a solution without a change output, Single Random Draw is an additional coin selection strategy besides the Knapsack solver. [This](https://github.com/bitcoin/bitcoin/pull/17526) strategy randomly picks OutputGroups(UTXOs with the same [scriptPubKey](https://learnmeabitcoin.com/technical/scriptPubKey)) from a pool of eligible UTXOs until the total amount is sufficient to cover the payment and fees. Any extra funds are put in a change output.

#### Pros of Single Random Draw
- Selected UTXOs have no consistent fingerprints like age or value
- Input sets are probabilistically representative of wallet's UTXO pool composition, becoming more consolidatory when more small UTXOs are present
- Easy to implement
- Random change output amounts increase value diversity

#### Cons of Single Random Draw
- Does not minimize fees
- May exhibit some fingerprints randomly such as revealing much greater funds in the wallet than necessary for payment


 It is important to note that there is a clear preference for the Branch and Bound coin selection strategy since if it is successful, no change output is produced. In case Branch and Bound fails ,Knapsack and Single Random Draw are then tried together, picking the solution with lower fees, breaking ties by number of UTXOs used.

## Conclusion

There are a lot of things that need to be considered when selecting coins. For example, by minimising change outputs, we make it harder for third parties to use on-chain analysis techniques that guess which output is the change returning excess funds from the input selection to the sender. When guessed correctly, this can undermine user privacy by clustering future transactions with the current one to build a wallet profile.

### References:
- https://bitcoincore.reviews/17331
- https://bitcoin.stackexchange.com/questions/32145/what-are-the-trade-offs-between-the-different-algorithms-for-deciding-which-utxo/32445#32445
- https://bitcoin.stackexchange.com/questions/1077/what-is-the-coin-selection-algorithm
- https://murch.one/posts/waste-metric/
- https://github.com/bitcoin/bitcoin/blob/master/src/wallet/spend.cpp#L526-L541


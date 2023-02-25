# Transaction malleability in Bitcoin: what is it and why it matters

Article structure

- Define problem set 
   - demo how hard it is to make btc micropayments onchain and how important it is to scale btc offchain
   - demo how difficult this scaling is since you have to depend on unconfirmed transactions
- Describe malleability
- Describe malleability solutions


One of the most important use cases for bitcoin is making payments at scale, big or small, across anywhere in the world at any time, instantly. This is a non-trivial problem because of all the trade-offs we have to balance. 





Thought experiment:
Scenarios......
Scenario 1

-- Give beneficiaries names and plausible use cases in real world to demonstrate point 


## So what exactly is transaction malleability?

The word malleate means to form—for example, metal with a hammer. This term is used in cryptography to mean changing a signature without making it invalid or changing an encrypted message without making it totally garbled.

Bitcoin transactions have a transaction id (txid) which is a hash of the data involved in the transaction. That suggests that it is a unique identifier for a transaction. However, the tx-id of a transaction is only unique once the exact data in the transaction has been added to a confirmed block, and block confirmation often takes some time. Until then, there are hacks that allow altering the underlying data and hash. It is important to note that it is not possible to alter how much bitcoin is transferred from what input to what output. But details such as what format the cryptogaphic signature confirming that the input authorizes the transaction takes, or the exact signature script used, can be changed: They are malleable, making the hash or transaction id malleable as well.


There are two kinds of transaction malleation: third party malleation, and first party malleation. In third party malleation, anyone can change a transaction's id by modifying parts of it that are not covered by a signature. This includes the signature itself. In first  party malleation, the transaction ID is modified by the sender(s). This is trivially done by re-signing the transaction but using a different nonce in the signature.

While malleation is modifying a transaction so its txid changes, a different way to view it is a transaction that double spends the original but happens to send the Bitcoin to the same outputs. This is how nodes and wallets view the transaction, there is nothing special to mark transactions as being malleated nor does there need to be, they just look like and should be treated as double spends. There is really no "original transaction" or "modified transaction", they are two distinct transactions that conflict with each other (spend the same inputs), that just so happen to have the same outputs.

## Consequences of transaction malleability

Transaction malleability can affect two major things: wallet software, and contracting protocols, for example the lightning network.

Altering one transaction id in the chain of unconfirmed transactions invalidates all successor transactions, until the sender resynchronizes with the blockchain rather than their own idea of which transaction ids they have sent. This is a big deal for bitcoin applications built on the assumption that transaction ids cannot be changed.Let's say we run a cryptocurrency exchange and we have to accept multiple customer withdrawals within a short time, and for record-keeping purposes, this exchange tracks transactions with transaction ids. Making multiple transactions directly building upon the unconfirmed previous ones is an easy mistake to make. If some of the exchange customers altered transaction ids for transactions being paid to them before onchain confirmation, and the exchange adopted such a rapid-fire approach to withdrawals, customers could falsely complain to the exchange that their withdrawals never materialised. The exchange cannot prove that the tx was actually made since the tweaked tx id on the blockchain and the tx id of the now-invalid transaction in the exchange's records do not match. That way, the exchange could then end up paying the customer more than once and incurring losses.

Malleability first became noticeable when wallet software would have issues because of malleated transactions. In the early days of Bitcoin, wallet software in general was not very good at handling double spends. This would result in issues such as wallets thinking they had access to more coins than they actually did, wallets continuously displaying and trying to rebroadcast conflicts that are no longer valid, and wallets trying to spend coins that no longer exist.

Transaction malleation was also noticed to be a major hindrance in contracting protocols such as Lightning. These protocols rely on the transaction ID not changing as they are pre-signing transactions that refer to a transaction that has not been broadcast yet. If the first transaction is malleated, and the counterparty is not cooperating (perhaps they malleated the first), then funds could be lost. Resolving these issues so that such protocols can work would require a solution that removes both third and first party malleation.


## Technical flaws leading to transaction malleability

The first flaw is that the original Bitcoin implementation used OpenSSL to verify the DER-encoded ASN.1 transaction data. However, OpenSSL did not do strict validation of the ASN.1 data by default. For instance, OpenSSL would ignore extra padding in the data. Just like adding trailing whitespace to a C file won't change the semantic meaning of the C code, Bob could add extra padding data to the transaction. This padding changes the transaction hash, just as adding trailing whitespace to a source code file would change the file hash.

The flaw related to DER-encoded ASN.1 data was fixed by the BIP66 soft fork. This became active on block 363,724 which was added to the blockchain on July 4, 2015. BIP66 is simple: it mandates a strict set of rules to how the ASN.1 data is encoded, and requires Bitcoin nodes to reject transactions that don't conform to the specification.

The second transaction malleability flaw was found later, and is much more subtle. The cryptographic signature scheme used by Bitcoin is ECDSA, which is a modified version of DSA using elliptic curves. An ECDSA signature consists of pair of numbers (r,s). The elliptic curve itself has integer order n. There is a surprising consequence of this, due to how the elliptic curve math works: if (r,s) is a valid signature, then so is the complementary signature (r,−s(mod n)). Given a signature (r,s) it's possible to calculate the complementary signature without knowing the ECDSA private keys. The complementary signature has a different hash, so using the complementary signature will result in a new transaction id. In other words, an attacker can change a txid by broadcasting a variation of the transaction that uses the complementary ECDSA signature.

In layman terms, one can produce a new valid signature for a transaction, given an existing valid one. That is, if Bob signs a transaction, anybody can produce another identical transaction (same inputs, outputs, etc), except with a different valid signature.

The fix for the ECDSA signing flaw is to enforce a canonical signature representation. Bitcoin core developers decided to use the following scheme: both signature values are calculated, but only the signature with the smaller "S-value" is considered valid. That is, the correct representation is the form with the smaller unsigned integer representation. The ECDSA signing flaw was originally supposed to be fixed by BIP62, which was later withdrawn. However, Bitcoin Core added a mechanism to enforce low S-values with PR #6769, which was merged in Bitcoin Core in October 2015. Validation is done when the transaction script contains the opcode SCRIPT_VERIFY_LOW_S, which all recent Bitcoin implementations use.

## How segwit fixed the problem

Segwit stands for “segregated witness,” and “witness” is just another word for the signature. This is a bitcoin upgrade that aimed to, among other things, move signature data out of transactions.

First we need to distinguish between legacy transactions(transaction formats before Segwit was activated) and Segwit transactions. For purposes of this explanation, a simplified version of a legacy Bitcoin transaction looks like this:

```
Tx: {
    Input: [(<previous TxID>,<tx output #>), ...],
    Output: [(<destination addresses>, <amount>), ...],
    Signature: [(<TxIDs>, <TxID’s relevant cryptographic signature(s)>)]
}
```

TxID stands for Transaction ID, and is the hash of some transaction in a block. Note that it’s not part of the transaction itself, but calculated based on the contents of the transaction.

Output is a list of amounts and destinations. It’s a list because a transaction can produce multiple outputs. For each output, there’s a controlling address–that’s who needs to sign any transaction that uses the coins in that output.

Input is a list of outputs from previous transactions. The bitcoin from those transactions will be spent in this transaction.

Signature is a list of the Signatures required to spend the transaction. Whoever owns the outputs of the previous transactions that are being used as the inputs to this one needs to use their private key to generate a Signature, saying they approve of their coins’ use.

Notice that the signature is part of the transaction in this format.


On the other hand, a Segwit transaction looks like this:

```
Tx: {
    Input: [(<previous TxID>,<tx output #>), ...],
    Output: [(<destination addresses>, <amount>), ...],
}
```

And somewhere else in the block, we have the signature of the transaction as follows:

```
Sig: {
    Signature: [(TxID, <Input TxID’s relevant cryptographic signature(s)>)]
}
```

Segwit made it so that signatures, scripts, and stack items are all stored in a separate part of the transaction which everyone would ignore for the purposes of transaction id calculation. It further does this only for a subset of scripts so that it can remain backwards compatible. In this way, instead of figuring out what could be malleable and explicitly enumerating them, it just takes everything that could be malleable and puts it somewhere else. However Segwit is not a perfect solution because:
- Things that were malleable previously still are malleable
- You have to opt-in to using Segwit and non-malleability 
- There are still ways to make segwit things malleable, but they basically require trying to make things that are malleable rather than malleability being the default as with non-segwit

Segwit transactions continue to include a legacy txid, but also include a wtxid field. The wtxid is calculated according to a strict set of rules over the transaction metadata, without including the ECDSA signature data when computing the transaction hash. This prevents all known transaction malleability attacks. Old clients can ignore the wtxid field and continue to use the legacy txid.

Note: Segwit transactions only avoid malleability if all their inputs are segwit spends (either directly, or via a backwards compatible segwit P2SH address).

### Who benefited from Segwit?

- Wallet authors tracking spent bitcoins: it’s easiest to monitor the status of your own outgoing transactions by simply looking them up by transaction id. But in a system with third-party malleability, wallets must implement extra code to be able to deal with changed txids.

 - Anyone spending unconfirmed transactions: if Alice pays Bob in transaction 1, Bob uses that payment to pay Charlie in transaction 2, and then Alice’s payment gets malleated and confirmed with a different transaction id, then transaction 2 is now invalid and Charlie has not been paid. If Bob is trustworthy, he will reissue the payment to Charlie; but if he isn’t, he can simply keep those bitcoins for himself.

- The Lightning Network: with third-party and scriptSig malleability fixed, the Lightning Network is less complicated to implement and significantly more efficient in its use of space on the blockchain. With scriptSig malleability removed, it also becomes possible to run lightweight Lightning clients that outsource monitoring the blockchain, instead of each Lightning client needing to also be a full Bitcoin node.

- Anyone using the block chain: smart contracts today, such as micropayment channels, and anticipated new smart contracts, become less complicated to design, understand, and monitor.

## Conclusion

Although transaction malleablity might not have been subtle to the existence of Bitcoin ,the fixing it was crucial to the scaling of the Bitcoin network and for increasing the adoption of Bitcoin worldwide. Segwit included a lot of improvements to Bitcoin ,and fixing transaction malleability was just one of those improvements. 

## References

- https://bitcoin.stackexchange.com/questions/21994/what-is-transaction-malleability
- https://www.ccn.com/mt-gox-blames-bitcoin-core-developer-greg-maxwell-responds/
- https://bitcointalk.org/index.php?topic=5440623.msg61794221#msg61794221
- https://bitcoin.stackexchange.com/questions/22051/transaction-malleability-in-the-blockchain/22058#22058
- https://eklitzke.org/bitcoin-transaction-malleability
- https://bitcoincore.org/en/2016/01/26/segwit-benefits/
- https://cowpig.github.io/bitcoin/cryptocurrency/2017/06/24/Segwit-and-Lightning-Network/

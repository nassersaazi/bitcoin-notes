https://eklitzke.org/bitcoin-transaction-malleability

- what is tx malleability?
- how it affected mt gox hack
- how it was fixed by segwit
- how fixing it with segwit enabled lightning

Who benefits?(develop case scenarios on these beneficiaries, then build article from there) -> kind of the style used here -> https://www.technologyreview.com/2023/02/14/1067869/rust-worlds-fastest-growing-programming-language/

    Wallet authors tracking spent bitcoins: it’s easiest to monitor the status of your own outgoing transactions by simply looking them up by txid. But in a system with third-party malleability, wallets must implement extra code to be able to deal with changed txids.

    Anyone spending unconfirmed transactions: if Alice pays Bob in transaction 1, Bob uses that payment to pay Charlie in transaction 2, and then Alice’s payment gets malleated and confirmed with a different txid, then transaction 2 is now invalid and Charlie has not been paid. If Bob is trustworthy, he will reissue the payment to Charlie; but if he isn’t, he can simply keep those bitcoins for himself.

    The Lightning Network: with third-party and scriptSig malleability fixed, the Lightning Network is less complicated to implement and significantly more efficient in its use of space on the blockchain. With scriptSig malleability removed, it also becomes possible to run lightweight Lightning clients that outsource monitoring the blockchain, instead of each Lightning client needing to also be a full Bitcoin node.

    Anyone using the block chain: smart contracts today, such as micropayment channels, and anticipated new smart contracts, become less complicated to design, understand, and monitor.

Note: segwit transactions only avoid malleability if all their inputs are segwit spends (either directly, or via a backwards compatible segwit P2SH address)


Is malleability really an issue?

This has to do with systems, such as Mt.Gox, that rely on unconfirmed transactions. Normal Bitcoin apps don't rely on unconfirmed transactions.


What malleability means is that you can't store the transaction ID that bitcoind returns from its sendtoaddress API call and expect that number to mean anything at all later.

Instead, if you want to keep track of a high volume of outgoing transactions, you have to wait for the transactions to be fully confirmed and immune to blockchain reorganizations, and then guess at which blockchain transactions correspond to which withdrawal or automatic payment events by matching the amount and destination address.

Some famous guy recommended using a combination of amount, address, and timestamp, but apparently transaction timestamps are not reliable either, so there really isn't a good, reliable way for Bitcoin apps to keep track of which transactions they sent out and why.

bitcoind itself tracks outputs, not transaction IDs, but it doesn't expose this information to API users. So if you want to send transactions and track them by the outputs, which is considered to be reliable, you need to write your own Bitcoin client, which is a lot like writing your own SQL server, except it's not as simple.


The Transaction Malleability issue that Mt.Gox talks about has been around since 2011. It only effects transactions which are not included in a block. Once a block is hashed, you can't change the transactions within it because then the hash of the block would have to change as well.

For all other Bitcoin applications that rely on confirmed transactions, as they should, this is not an issue
From Quartz news article about tx malleability

A bug in the bitcoin software makes it possible for someone to use the Bitcoin network to alter transaction details to make it seem like a sending of bitcoins to a bitcoin wallet did not occur when in fact it did occur…This defect, known as “transaction malleability” makes it possible for a third party to alter the hash of any freshly issued transaction without invalidating the signature, hence resulting in a similar transaction under a different hash.

Definition
A property of Bitcoin transactions that allows them to be replicated with another transaction id before they are included in a block.

Bitcoin transactions have a transaction id (txid) formed as a hash over the data involved in the transaction. That suggests that it is a unique identifier for a transaction.

However, the txid of a transaction is only unique once the exact data in the transaction has been finalized by being incorporated into the blockchain (and confirmed). Until then, there are hacks that allow altering the underlying data and hash. This is not a security issue because it is not possible to alter how many bitcoins are transferred from what input to what output. But details such as what format the transaction’s cryptographic signature takes, or the exact signature script used, can be changed: They are malleable, making the derived hash or transaction id malleable as well.

b4 segwit
While transactions are signed, the signature does not currently cover all the data in a transaction that is hashed to create the transaction hash. Thus, while uncommon, it is possible for a node on the network to change a transaction you send in such a way that the hash is invalidated. Note that this just changes the hash; the output of the transaction remains the same and the bitcoins will go to their intended recipient. However this does mean that, for instance, it is not safe to accept a chain of unconfirmed transactions under any circumstance because the later transactions will depend on the hashes of the previous transactions, and those hashes can be changed until they are confirmed in a block (and potentially even after a confirmation if the block chain is reorganized). In addition, clients must always actively scan for transactions to them; assuming a txout exists because the client created it previously is unsafe

BIP_0062 was a bitcoin improvement proposal from early 2014 which aimed to deal with malleability. It aimed to find all possible methods of malleability and fix them one-by-one. The BIP was withdrawn because that was found to be not sufficient for the use cases which malleability prevents (e.g payment channels).[4][5][6] The BIP document itself contains all malleability methods that the authors could think of.

Segregated Witness was an update to bitcoin had as one of its aims to fix all forms of malleability. Transactions which only spend segwit outputs are not vulnerable to malleability

Bitcoin payments are encoded as transactions that eventually become part of the blockchain. Each Bitcoin transaction contains metadata such as: the input addresses (where the money is coming from), the output addresses (where the money is going), the amount of Bitcoin actually being sent, and cryptographic signatures proving the authenticity of the transaction. This data is bundled into a DER-encoded ASN.1 representation before being broadcast to the network. Each transaction has a "transaction id" or txid, which is a hash of the transaction. An example txid is adae0270457bad95152c5ae7771b50fae06afa01edeefca4201689e7c99e0b19. This hexadecimal string is calculated using a variant of SHA-256 on the DER-encoded transaction data.

These txids are immaterial to how the Bitcoin blockchain works: their primary use is as a convenience for humans when referring to transactions. For instance, suppose you want to buy something online, and send a Bitcoin payment to an ecommerce site. If there's a problem with the merchant's ecommerce software, it's possible that they could "lose" the transaction, meaning they might think you haven't actually paid them. You could then show them the txid of your payment, and then the merchant could then manually reconcile the error after confirming the transaction. For this to work the txids need to be immutable, and that was the original intention in Bitcoin.

Bitcoin is a peer-to-peer network, operating using a gossip protocol which is conceptually similar to BitTorrent. To send a payment, a node creates a transaction and then broadcasts it to the node's peers on the network. The peers then broadcast the transaction to their peers, and so on. Usually it takes less than a minute from the time a transaction is created until it fully propagates to the rest of the network. Well connected nodes in Europe and North America have typical propagation times on the order of 10 to 15 seconds.

Here's how the transaction malleability attack works. Alice creates a Bitcoin payment transaction, and sends it to her peers. The original Bitcoin implementation was underspecified with respect to how txids were actually calculated (more on this in a moment). Therefore, it's possible for Alice's peers to slightly modify the transaction. Suppose Bob is a peer of Alice, and wants to initiate a transaction malleability attack against Alice. The inputs, outputs, and payment amount are all cryptographically signed, so Bob can't steal money or make any semantic changes to the transaction. However, Bob can make some changes that don't change the transaction semantics, but do change the computed txid. At this point Bob will broadcast the transaction with a new txid to the rest of the network. At this point it's a race to see which transaction will actually be accepted by the network: the original transaction created by Alice and relayed by her good peers, or the modified version created by Bob. The attack is called "transaction malleability" because Bob was able to modify the transaction, even though the transaction was supposed to be immutable.

Transaction Malleability Significance

Most Bitcoin clients have an option to show you a txid after you send a transaction. Bitcoin transactions take some time to actually be confirmed as part of the blockchain. Therefore it's natural to periodically check the blockchain to see if the transaction has actually gone through, by checking if the expected txid has been added to a new block. If a transaction malleability attack occurs, and the txid changes, then the transaction will eventually be added to the blockchain, but under an unexpected txid. This can confuse client software that was looking for a particular txid.

For instance, suppose Alice sends 1 BTC with expected txid A, but Bob modifies the transaction so the new txid is B. The modified transaction B then gets added to the blockchain, which implicitly invalidates A. Alice's client software keeps checking for txid A, but never sees it. Alice's wallet software will debit 1 BTC from her account once the modified transaction is confirmed, since the modified transaction still sent 1 BTC from her account. But if Alice isn't paying close attention, she might eventually give up and think the transaction failed for some reason, and she could retry the transaction. If she does retry the transaction, she'll send another 1 BTC to the same address. In essence, Bob has tricked Alice into double paying


It is thought that this attack was used against some Bitcoin exchanges, including Mt Gox. Here's how it would work. You deposit 1 BTC into an account on an exchange. Later, you try to withdraw your 1 BTC off the exchange, back to your private wallet. If you control nodes that peer with the exchange, you might be able to change the txid for your withdrawal using transaction malleability. The 1 BTC you withdrew will go into your private wallet under a new txid. If the exchange is naive, you might be able to trick the exchange into thinking that it never sent you your withdrawal. Then you'd ask to withdraw your 1 BTC again, and if you tricked the exchange it could comply. By doing this repeatedly, you could potentially withdraw a large amount of Bitcoin before the exchange caught on. This is possibly what happened to Mt Gox (but see below for a more detailed analysis).

How Transaction Malleability Actually Works

Before continuing, I want to re-emphasize that Bob can't change where Alice's money comes from, where it goes, or how much is sent. These parameters are all cryptographically signed by Alice, using her private key. Bob can really just change the actual txid shown to humans. How does this work exactly?

The first flaw is that the original Bitcoin implementation used OpenSSL to verify the DER-encoded ASN.1 transaction data. However, OpenSSL did not do strict validation of the ASN.1 data by default. For instance, OpenSSL would ignore extra padding in the data. Just like adding trailing whitespace to a C file won't change the semantic meaning of the C code, Bob could add extra padding data to the transaction. This padding changes the transaction hash, just as adding trailing whitespace to a source code file would change the file hash.

The flaw related to DER-encoded ASN.1 data was fixed by the BIP66 soft fork. This became active on block 363,724 which was added to the blockchain on July 4, 2015. BIP66 is simple: it mandates a strict set of rules to how the ASN.1 data is encoded, and requires Bitcoin nodes to reject transactions that don't conform to the specification.

The second transaction malleability flaw was found later, and is much more subtle. The cryptographic signature scheme used by Bitcoin is ECDSA, which is a modified version of DSA using elliptic curves. An ECDSA signature consists of pair of numbers (r,s). The elliptic curve itself has integer order n. There is a surprising consequence of this, due to how the elliptic curve math works: if (r,s) is a valid signature, then so is the complementary signature (r,−smodn). Given a signature (r,s) it's possible to calculate the complementary signature without knowing the ECDSA private keys. The complementary signature has a different hash, so using the complementary signature will result in a new txid. In other words, an attacker can change a txid by broadcasting a variation of the transaction that uses the complementary ECDSA signature.

The fix for the ECDSA signing flaw is to enforce a canonical signature representation. The Bitcoin core developers decided to use the following scheme: both signature values are calculated, but only the signature with the smaller "S-value" is considered valid. That is, the correct representation is the form with the smaller unsigned integer representation. The ECDSA signing flaw was originally supposed to be fixed by BIP62, which was later withdrawn. However, Bitcoin Core added a mechanism to enforce low S-values with PR #6769, which was merged in Bitcoin Core in October 2015. Validation is done when the transaction script contains the opcode SCRIPT_VERIFY_LOW_S, which all recent Bitcoin implementations use.
The Future

I'd be remiss to end this post without mentioning the hottest topic in Bitcoin right now, BIP141 a.k.a. Segregated Witness or "Segwit". Transaction malleability is already more or less fixed in Bitcoin, but Segwit will improve the situation further with the introduction of a new type of txid, the wtxid (i.e. "witness" txid).

Here's how Segwit fixes the problem. When we think of a transaction, we really just care about the inputs, outputs, and payment amounts. ECDSA signatures are essential to the Bitcoin security model, but don't actually affect these transaction details. Segwit transactions continue to include a legacy txid as described here, but also include a new wtxid field. The wtxid is calculated according to a strict set of rules over the transaction metadata, without including the ECDSA signature data when computing the transaction hash. This prevents all known transaction malleability attacks. Old clients can ignore the wtxid field and continue to use the legacy txid.

Fixing transaction malleability is just one aspect of Segwit. BIP141 has a number of other improvements as well: it makes a number of significant changes to the Bitcoin scripting language, and will enable the use of cryptographically secure off-chain transaction using the Lightning Network. BIP141 should make future protocol extensions to Bitcoin much easier to deploy. As of the time of this writing, the most likely scenario is that Segwit will get "locked in" later this month, and then activate sometime in August.

Postscript: The Mt Gox Hack

Since the Mt Gox incident is so famous, I want to point out that it's not known definitively if transaction malleability is actually what caused Mt Gox to become insolvent. Mark Karpeles, the founder of Mt Gox, claimed that coins were stolen using this flaw, but it's hard to independently verify this claim. An attacker exploiting transaction malleability could generate new receive addresses for each transaction. They could also randomize the withdrawal amounts. This would make it very difficult (potentially impossible) to audit the blockchain to verify that this attack was used against Mt Gox. To verify the claim, you'd need the actual Mt Gox database records to perform a full analysis. It's possible that only a small percentage of stolen coins from Mt Gox were taken using this attack, or even none at all.

Regardless of what happened with Mt Gox, since this transaction malleability is quite subtle, it's likely that other exchanges or sites were tricked into resending payments using transaction malleability.

references:
- https://bitcoin.stackexchange.com/questions/21994/what-is-transaction-malleability
- https://qz.com/175565/why-nobody-can-withdraw-bitcoins-from-one-of-the-currencys-largest-exchanges
- https://qz.com/154877/by-reading-this-page-you-are-mining-bitcoins
- https://qz.com/175565/why-nobody-can-withdraw-bitcoins-from-one-of-the-currencys-largest-exchanges
- https://www.ccn.com/mt-gox-blames-bitcoin-core-developer-greg-maxwell-responds/
- http://diyhpl.us/wiki/transcripts/scalingbitcoin/hong-kong/overview-of-bips-necessary-for-lightning/
- https://bitcoinmagazine.com/culture/the-who-what-why-and-how-of-the-ongoing-transaction-malleability-attack-1444253640
- https://bitcoin.stackexchange.com/questions/22051/transaction-malleability-in-the-blockchain/22058#22058
- https://eklitzke.org/bitcoin-transaction-malleability

todos:
- write twitter thread on tx malleability demonstrating role of btc devs in fixing it ,and how critical devs are in the bitcoin existence puzzle. refer to @jamesob's tweet about dev funding worries

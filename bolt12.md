- write about bolt 12 after malleability article -> refer to (https://bolt12.org/)

- define current problem with bolt 11
- describe how bolt 12 attempts to fix it
- do a coding exercise to set up cln on signet and transact with a bolt 12 invoice


Limitations of BOLT 11

The BOLT 11 invoice format has proven popular, but has several limitations:

    The entangling of bech32 encoding makes it awkward to send in other forms (e.g. inside the lightning network itself).
    The signature applying to the entire invoice makes it impossible to prove an invoice without revealing its entirety.
    Fields cannot generally be extracted for external use: the h field was a boutique extraction of the d field, only.
    The lack of 'it's OK to be odd' rule makes backwards compatibility harder.
    The 'human-readable' idea of separating amounts proved fraught: p was often mishandled, and amounts in pico-bitcoin are harder than the modern satoshi-based counting.
    The bech32 encoding was found to have an issue with extensions, which means we want to replace or discard it anyway.
    The payment_secret designed to prevent probing by other nodes in the path was only useful if the invoice remained private between the payer and payee.
    Invoices must be given per-user, and are actively dangerous if two payment attempts are made for the same user.

Payment Flow Scenarios

Here we use "user" as shorthand for the individual user's lightning node, and "merchant" as the shorthand for the node of someone who is selling or has sold something.

There are two basic payment flows supported by BOLT 12:

The general user-pays-merchant flow is:

    A merchant publishes an offer ("send me money"), such as on a web page or a QR code.
    Every user requests a unique invoice over the lightning network using an invoice_request message.
    The merchant replies with the invoice.
    The user makes a payment to the merchant indicated by the invoice.

The merchant-pays-user flow (e.g. ATM or refund):

    The merchant provides a user-specific offer ("take my money") in a web page or QR code, with an amount (for a refund, also a reference to the to-be-refunded invoice).
    The user sends an invoice for the amount in the offer (for a refund, a proof that they requested the original)
    The merchant makes a payment to the user indicated by the invoice.

Payment Proofs and Payer Proofs

Note that the normal lightning "proof of payment" can only demonstrate that an invoice was paid (by showing the preimage of the payment_hash), not who paid it. The merchant can claim an invoice was paid, and once revealed, anyone can claim they paid the invoice, too.[1]

Providing a key in invoice_request allows a user to prove that they were the one to request the invoice. In addition, the Merkle construction of the BOLT 12 invoice signature allows the user to selectively reveal fields of the invoice in case of dispute.
Encoding

Each of the forms documented here are in TLV format.

The supported ASCII encoding is the human-readable prefix, followed by a 1, followed by a bech32-style data string of the TLVs in order, optionally interspersed with + (for indicating additional data is to come).
Requirements

Readers of a bolt12 string:

    if it encounters a + followed zero or more whitespace characters between two bech32 characters:
        MUST remove the + and whitespace.

Rationale

The use of bech32 is arbitrary, but already exists in the bitcoin world. We currently omit the six-character trailing checksum: QR codes have their own checksums, bech32 doesn't protect against many length differences, and bech32m is not yet widely supported.

The use of + (which is ignored) allows use over limited text fields like Twitter:

lno1xxxxxxxx+

yyyyyyyyyyyy+

zzzzz

See format-string-test.json.

Signature Calculation

All signatures are created as per BIP-340, and tagged as recommended there. Thus we define H(tag,msg) as SHA256(SHA256(tag) || SHA256(tag) || msg), and SIG(tag,msg,key) as the signature of H(tag,msg) using key.

Each form is signed using one or more TLV signature elements; TLV types 240 through 1000 are considered signature elements. For these the tag is "lightning" || messagename || fieldname, and msg is the Merkle-root; "lightning" is the literal 9-byte ASCII string, messagename is the name of the TLV stream being signed (i.e. "offer", "invoice_request" or "invoice") and the fieldname is the TLV field containing the signature (e.g. "signature" or "payer_signature").

The formulation of the Merkle tree is similar to that proposed in [BIP-taproot], with each TLV leaf paired with a nonce leaf to avoid revealing adjacent nodes in proofs (assuming there is a non-revealed TLV which has enough entropy).

The Merkle tree's leaves are, in TLV-ascending order for each tlv:

    The H(LnLeaf,tlv).
    The H(LnAll||all-tlvs,tlv) where "all-tlvs" consists of all non-signature TLV entries appended in ascending order.

The Merkle tree inner nodes are H(LnBranch, lesser-SHA256||greater-SHA256); this ordering means that proofs are more compact since left/right is inherently determined.

If there are not exactly a power of 2 leaves, then the tree depth will be uneven, with the deepest tree on the lowest-order leaves

e.g. consider the encoding of an offer signature with TLVs TLV1, TLV2 and TLV3:

L1=H("LnLeaf",TLV1)
L1nonce=H("LnAll"||TLV1||TLV2||TLV3,TLV1)
L2=H("LnLeaf",TLV2)
L2nonce=H("LnAll"||TLV1||TLV2||TLV3,TLV2)
L3=H("LnLeaf",TLV3)
L3nonce=H("LnAll"||TLV1||TLV2||TLV3,TLV3)

Assume L1 < L1nonce, and L2 > L2nonce.
```
   L1    L1nonce                      L2   L2nonce                L3   L3nonce
     \   /                             \   /                       \   /
      v v                               v v                         v v
L1A=H("LnBranch",L1||L1nonce) L2A=H("LnBranch",L2nonce||L2)  L3A=H("LnBranch",L3nonce||L3)

Assume L1A < L2A:

       L1A   L2A                                 L3A=H("LnBranch",L3nonce||L3)
         \   /                                    |
          v v                                     v
  L1A2A=H("LnBranch",L1A||L2A)                   L3A=H("LnBranch",L3nonce||L3)

Assume L1A2A > L3A:

  L1A2A=H("LnBranch",L1A||L2A)          L3A
                          \            /
                           v          v
                Root=H("LnBranch",L3A||L1A2A)

Signature = SIG("lightningoffersignature", Root, nodekey)
```

Offers

Offers are a precursor to an invoice: readers will either request an invoice (or multiple) or send an invoice based on the offer. An offer can be much longer-lived than a particular invoice, so has some different characteristics; in particular it can be recurring, and the amount can be in a non-lightning currency. It's also designed for compactness, to easily fit inside a QR code.

The human-readable prefix for offers is lno.

TLV Fields for Offers

    tlv_stream: offer

    types:
        type: 2 (chains)
        data:
            [...*chain_hash:chains]
        type: 6 (currency)
        data:
            [...*utf8:iso4217]
        type: 8 (amount)
        data:
            [tu64:amount]
        type: 10 (description)
        data:
            [...*utf8:description]
        type: 12 (features)
        data:
            [...*byte:features]
        type: 14 (absolute_expiry)
        data:
            [tu64:seconds_from_epoch]
        type: 16 (paths)
        data:
            [...*blinded_path:paths]
        type: 20 (issuer)
        data:
            [...*utf8:issuer]
        type: 22 (quantity_min)
        data:
            [tu64:min]
        type: 24 (quantity_max)
        data:
            [tu64:max]
        type: 30 (node_id)
        data:
            [point32:node_id]
        type: 54 (send_invoice)
        type: 34 (refund_for)
        data:
            [sha256:refunded_payment_hash]
        type: 240 (signature)
        data:
            [bip340sig:sig]

    subtype: blinded_path
    data:
        [point:first_node_id]
        [point:blinding]
        [byte:num_hops]
        [num_hops*onionmsg_path:path]

Requirements For Offers

A writer of an offer:

    MUST set node_id to the public key of the node to request the invoice from.
    MAY specify exactly one signature TLV: signature:
        If so, it MUST set sig to the signature using node_id as described in Signature Calculation.
    MUST set description to a complete description of the purpose of the payment.
    if the chain for the invoice is not solely bitcoin:
        MUST specify chains the offer is valid for.
    otherwise:
        the bitcoin chain is implied as the first and only entry.
    if a specific minimum amount is required for successful payment:
        MUST set amount to the amount expected (per item).
        if the currency for amount is that of the first entry in chains:
            MUST specify amount in multiples of the minimum lightning-payable unit (e.g. milli-satoshis for bitcoin).
        otherwise:
            MUST specify iso4217 as an ISO 4712 three-letter code.
            MUST specify amount in the currency unit adjusted by the ISO 4712 exponent (e.g. USD cents).
    if it supports offer features:
        SHOULD set features to the bitmap of offer features.
    if the offer expires:
        MUST set absolute_expiry seconds_from_epoch to the number of seconds after midnight 1 January 1970, UTC that invoice_request should not be attempted.
    if it is connected only by private channels:
        MUST include paths containing one or more paths to the node from publicly reachable nodes.
    otherwise:
        MAY include paths.
    if it includes paths:
        SHOULD ignore any invoice_request which does not use the path.
    if it sets issuer:
        SHOULD set it to clearly identify the issuer of the invoice.
        if it includes a domain name:
            SHOULD begin it with either user@domain or domain
            MAY follow with a space and more text
    if it can supply more than one item for a single invoice
        if the minimum quantity is more than 1:
            MUST set that minimum in quantity_min
        if the maximum quantity is known:
            MUST set that maximum in quantity_max
        if neither:
            MUST set quantity_min to 1 to indicate quantity is supported.
        if both:
            MUST set quantity_min less than or equal to quantity_max.
        MUST NOT set quantity_min or quantity_max less than 1.
    if send_invoice is present:
        if the offer is for a partial or full refund for a previously-paid invoice:
            SHOULD set refunded_payment_hash to the payment_hash of that invoice.
    otherwise:
        MUST NOT set refunded_payment_hash.

A reader of an offer:

    if features contains unknown odd bits that are non-zero:
        MUST ignore the bit.
    if features contains unknown even bits that are non-zero:
        MUST NOT respond to the offer.
        SHOULD indicate the unknown bit to the user.
    if node_id or description is not set:
        MUST NOT respond to the offer.
    if signature is present, but is not a valid signature using node_id as described in Signature Calculation:
        MUST NOT respond to the offer.
    SHOULD gain user consent for recurring payments.
    SHOULD allow user to view and cancel recurring payments.
    if it uses amount to provide the user with a cost estimate:
        MUST warn user if amount of actual invoice differs significantly from that expectation.
    SHOULD not respond to an offer if the current time is after absolute_expiry.
    FIXME: more!

Rationale

A signature is optional, because it makes for a longer string (potentially limiting QR code use on low-end cameras); if the offer has an error, no invoice will be given (or, for send_invoice offers, accepted), since the offer_id already covers all the non-signature fields.

Invoice Requests

Invoice Requests are a request for an invoice; the human-readable prefix for invoices is lnr.
TLV Fields for invoice_request

    tlv_stream: invoice_request
    types:
        type: 3 (chain)
        data:
            [chain_hash:chain]
        type: 4 (offer_id)
        data:
            [sha256:offer_id]
        type: 8 (amount)
        data:
            [tu64:msat]
        type: 12 (features)
        data:
            [...*byte:features]
        type: 32 (quantity)
        data:
            [tu64:quantity]
        type: 38 (payer_key)
        data:
            [point32:key]
        type: 39 (payer_note)
        data:
            [...*utf8:note]
        type: 50 (payer_info)
        data:
            [...*byte:blob]
        type: 56 (replace_invoice)
        data:
            [sha256:payment_hash]
        type: 240 (payer_signature)
        data:
            [bip340sig:sig]

Requirements for Invoice Requests

The writer of an invoice_request:

    MUST set payer_key to a transient public key.
    MUST remember the secret key corresponding to payer_key.
    MUST set offer_id to the Merkle root of the offer as described in Signature Calculation.
    MUST NOT set or imply any chain_hash not set or implied by the offer.
    MUST set payer_signature sig as detailed in Signature Calculation using the payer_key.
    if the offer had a quantity_min or quantity_max field:
        MUST set quantity
        MUST set it within that (inclusive) range.
    otherwise:
        MUST NOT set quantity
    if the offer did not specify amount:
        MUST specify amount.msat in multiples of the minimum lightning-payable unit (e.g. milli-satoshis for bitcoin) for chain (or for bitcoin, if there is no chain).
    otherwise:
        MAY omit amount.
        if it sets amount:
            MUST specify amount.msat as greater or equal to amount expected by the offer (before any proportional period amount).
    if the sender has a previous unpaid invoice (for the same offer) which it wants to cancel:
        MUST set payer_key to the same as the previous invoice.
        MUST set replace_invoice to the payment_hash or the previous invoice.

The reader of an invoice_request:

    MUST fail the request if payer_key is not present.
    if chain is not present:
        MUST fail the request if bitcoin is not a supported chain.
    otherwise:
        MUST fail the request if chain is not a supported chain.
    MUST fail the request if features contains unknown even bits.
    MUST fail the request if offer_id is not present.
    MUST fail the request if the offer_id does not refer to an unexpired offer.
    MUST fail the request if there is no payer_signature field.
    MUST fail the request if payer_signature is not correct.
    if the offer had a quantity_min or quantity_max field:
        MUST fail the request if there is no quantity field.
        MUST fail the request if there is quantity is not within that (inclusive) range.
    otherwise:
        MUST fail the request if there is a quantity field.
    if the offer included amount:
        MUST calculate the base invoice amount using the offer amount:
            if offer currency is not the invoice currency, convert to the invoice currency.
            if request contains quantity, multiply by quantity.
        if the request contains amount:
            MUST fail the request if its amount is less than the base invoice amount.
            MAY fail the request if its amount is much greater than the base invoice amount.
            MUST use the request's amount as the base invoice amount.
    otherwise:
        MUST fail the request if it does not contain amount.
        MUST use the request amount as the base invoice amount.
    if the offer has a replace_invoice:
        if the payment_hash refers to an unpaid invoice for the same offer_id and payer_key:
            MUST immediately expire/remove that unpaid invoice such that it cannot be paid in future.
        otherwise:
            MUST fail the request.

Rationale

payer_info might typically contain information about the derivation of the payer_key. This should not leak any information (such as using a simple BIP-32 derivation path); a valid system might be for a node to maintain a base payer key, and encode a 128-bit tweak here. The payer_key would be derived by tweaking the base key with SHA256(payer_base_pubkey || tweak).

payer_note allows you to compliment, taunt, or otherwise engrave graffiti into the invoice for all to see.

Users can give a tip (or obscure the amount sent) by specifying an amount in their invoice request, even though the offer specifies an amount. Obviously this will only be accepted by the recipient if the invoice request amount exceeds the amount it's expecting (i.e. its amount after any currency conversion, multiplied by quantity if any). Note that for recurring invoices with proportional_amount set, the amount in the invoice request will be scaled by the time in the period; the sender should not attempt to scale it.

replace_invoice allows the mutually-agreed removal of and old unpaid invoice; this can be used in the case of stuck payments. If successful in replacing the stuck invoice, the sender may make a second payment such that it can prove double-payment should the receiver still accept the first, delayed payment


Invoices

Invoices are a request for payment, and when the payment is made it can be combined with the invoice to form a cryptographic receipt.

The human-readable prefix for invoices is lni. It can be sent in response to an invoice_request or an offer with send_invoice using onion_message invoice field.

    tlv_stream: invoice

    types:
        type: 3 (chain)
        data:
            [chain_hash:chain]
        type: 4 (offer_id)
        data:
            [sha256:offer_id]
        type: 8 (amount)
        data:
            [tu64:msat]
        type: 10 (description)
        data:
            [...*utf8:description]
        type: 12 (features)
        data:
            [...*byte:features]
        type: 16 (paths)
        data:
            [...*blinded_path:paths]
        type: 18 (blindedpay)
        data:
            [...*blinded_payinfo:payinfo]
        type: 19 (blinded_capacities)
        data:
            [...*u64:incoming_msat]
        type: 20 (issuer)
        data:
            [...*utf8:issuer]
        type: 30 (node_id)
        data:
            [point32:node_id]
        type: 32 (quantity)
        data:
            [tu64:quantity]
        type: 34 (refund_for)
        data:
            [sha256:refunded_payment_hash]
        type: 38 (payer_key)
        data:
            [point32:key]
        type: 39 (payer_note)
        data:
            [...*utf8:note]
        type: 50 (payer_info)
        data:
            [...*byte:blob]
        type: 40 (created_at)
        data:
            [tu64:timestamp]
        type: 42 (payment_hash)
        data:
            [sha256:payment_hash]
        type: 44 (relative_expiry)
        data:
            [tu32:seconds_from_creation]
        type: 46 (cltv)
        data:
            [tu32:min_final_cltv_expiry]
        type: 48 (fallbacks)
        data:
            [byte:num]
            [num*fallback_address:fallbacks]
        type: 52 (refund_signature)
        data:
            [bip340sig:payer_signature]
        type: 56 (replace_invoice)
        data:
            [sha256:payment_hash]
        type: 240 (signature)
        data:
            [bip340sig:sig]

    subtype: blinded_payinfo

    data:
        [u32:fee_base_msat]
        [u32:fee_proportional_millionths]
        [u16:cltv_expiry_delta]
        [u16:flen]
        [flen*byte:features]

    subtype: fallback_address
    data:
        [byte:version]
        [u16:len]
        [len*byte:address]

Requirements

A writer of an invoice:

    MUST set created_at to the number of seconds since Midnight 1 January 1970, UTC when the offer was created.
    MUST set payment_hash to the SHA2 256-bit hash of the payment_preimage that will be given in return for payment.
    MUST set (or not set) send_invoice the same as the offer.
    MUST set offer_id to the id of the offer.
    MUST specify exactly one signature TLV: signature.
        MUST set sig to the signature using node_id as described in Signature Calculation.
    if the chain for the invoice is not bitcoin:
        MUST specify chain the invoice is valid for.
    otherwise:
        the bitcoin chain is implied as the first and only entry.
    if it has bolt11 features:
        MUST set features to the bitmap of features.
    if the expiry for accepting payment is not 7200 seconds after created_at:
        MUST set relative_expiry seconds_from_creation to the number of seconds after created_at that payment of this invoice should not be attempted.
    if the min_final_cltv_expiry for the last HTLC in the route is not 18:
        MUST set min_final_cltv_expiry.
    if it accepts onchain payments:
        MAY specify fallbacks
        MUST specify fallbacks in order of most-preferred to least-preferred if it has a preference.
        for the bitcoin chain, it MUST set each fallback_address with version as a valid witness version and address as a valid witness program
    if it is connected only by private channels:
        MUST include a blinded_path containing one or more paths to the node.
    otherwise:
        MAY include blinded_path.
    if it includes blinded_path:
        MUST specify path in order of most-preferred to least-preferred if it has a preference.
        MUST include blindedpay with exactly one payinfo for each onionmsg_path in blinded_path, in order.
        if it includes blinded_capacities:
            MUST include exactly one incoming_msat (in millisatoshis) per path, reflecting the expected maximum amount that can be sent through the path.
        SHOULD ignore any payment which does not use one of the paths.
    otherwise:
        MUST NOT include blinded_payinfo.
    MUST specify amount.msat in multiples of the minimum lightning-payable unit (e.g. milli-satoshis for bitcoin) for chain (or for bitcoin, if there is no chain).
    if responding to an invoice_request:
        if for the same offer_id and payer_key as a previous invoice_request:
            MAY simply reuse the previous invoice.
        otherwise:
            MUST NOT reuse a previous invoice.
        MUST set node_id the same as the offer.
        MUST set (or not set) quantity exactly as the invoice_request did.
        MUST set payer_key exactly as the invoice_request did.
        MUST set (or not set) payer_info exactly as the invoice_request did.
        MUST set (or not set) payer_note exactly as the invoice_request did, or MUST not set it.
        MUST set (or not set) replace_invoice exactly as the invoice_request did.
        MUST begin description with the description from the offer.
        MAY append additional information to description (e.g. " +shipping").
        if it does not set amount to the base invoice amount calculated from the invoice_request:
            MUST append the reason to description (e.g. " 5% bulk discount").
        MUST set (or not set) issuer exactly as the offer did.
        MUST NOT set refund_for
        MUST NOT set refund_signature
    otherwise (responding to a send_invoice offer):
        MUST set node_id to the id of the node to send payment to.
        MUST set description the same as the offer.
        if the offer had a quantity_min or quantity_max field:
            MUST set quantity
            MUST set it within that (inclusive) range.
        otherwise:
            MUST NOT set quantity
        MUST set payer_key to the node_id of the offer.
        MUST NOT set payer_info.
        MUST set (or not set) refund_for exactly as the offer did.
        if it sets refund_for:
            MUST set refund_signature to the signature of the refunded_payment_hash using prefix refund_signature and the payer_key from the to-be-refunded invoice.
        otherwise:
            MUST NOT set refund_signature

A reader of an invoice:

    MUST reject the invoice if signature is not a valid signature using node_id as described in Signature Calculation.
    MUST reject the invoice if msat is not present.
    MUST reject the invoice if description is not present.
    MUST reject the invoice if created_at is not present.
    MUST reject the invoice if payment_hash is not present.
    if relative_expiry is present:
        MUST reject the invoice if the current time since 1970-01-01 UTC is greater than created_at plus seconds_from_creation.
    otherwise:
        MUST reject the invoice if the current time since 1970-01-01 UTC is greater than created_at plus 7200.
    if blinded_path is present:
        MUST reject the invoice if blinded_payinfo is not present.
        MUST reject the invoice if blinded_payinfo does not contain exactly as many payinfo as total onionmsg_path in blinded_path.
    SHOULD confirm authorization if msat is not within the amount range authorized.
    if the invoice is a reply to an invoice_request:
        MUST reject the invoice unless offer_id is equal to the id of the offer.
        MUST reject the invoice unless node_id is equal to the offer.
        MUST reject the invoice unless the following fields are equal or unset exactly as they are in the invoice_request:
            quantity
            payer_key
            payer_info
            MUST reject the invoice if payer_note is set, and was unset or not equal to the field in the invoice_request.
            SHOULD confirm authorization if the description does not exactly match the offer
            MAY highlight if description has simply had a change appended.
            SHOULD confirm authorization if issuer does not exactly match the offer.
    otherwise if offer_id is set:
        MUST reject the invoice if the offer_id does not refer an unexpired offer with send_invoice
        MUST reject the invoice unless the following fields are equal or unset exactly as they are in the offer:
            refund_for
            description
        if the offer had a quantity_min or quantity_max field:
            MUST reject the invoice if there is no quantity field.
            MUST reject the invoice if there is quantity is not within that (inclusive) range.
        otherwise:
            MUST reject the invoice if there is a quantity field.
    if the offer contained refund_for:
        MUST reject the invoice if payer_key does not match the invoice whose payment_hash is equal to refund_for refunded_payment_hash
        MUST reject the invoice if refund_signature is not set.
        MUST reject the invoice if refund_signature is not a valid signature using payer_key as described in Signature Calculation.
    for the bitcoin chain, if the invoice specifies fallbacks:
        MUST ignore any fallback_address for which version is greater than 16.
        MUST ignore any fallback_address for which address is less than 2 or greater than 40 bytes.
        MUST ignore any fallback_address for which address does not meet known requirements for the given version
    if the min_final_cltv_expiry is specified:
        MUST use an expiry delta of at least that value when making the payment
    otherwise:
        MUST use an expiry delta of at least 18 when making the payment

Rationale

Because the messaging layer is unreliable, it's quite possible to receive multiple requests for the same offer. As it's the caller's responsibility not to reuse payer_key except for recurring invoices, the writer doesn't have to check all the fields are duplicates before simply returning a previous invoice. Note that such caching is optional, and should be carefully limited when e.g. currency conversion is involved, or if the invoice has expired.

The invoice duplicates fields rather than committing to the previous offer or invoice_request. This flattened format simplifies storage at some space cost, as the payer need only remember the invoice for any refunds or proof.

The reader of the invoice cannot trust the invoice correctly reflects the offer and invoice_request fields, hence the requirements to check that they are correct.

Note that the recipient of the invoice can determine the expected amount from either the offer it received, or the invoice_request it sent, so often already has authorization for the expected amount.

It's natural to set the relative_expiry of an invoice for a recurring offer to the end of the payment window for the period, but if that is a long time and the offer was in another currency, it's common to cap this at some maximum duration. For example, omitting it implies the default of 7200 seconds, which is generally a sufficient time for payment.

The invoice issuer is allowed to ignore payer_note (it has an odd number, so is optional), but if it does not, it must copy it exactly as the invoice_request specified.

It's often useful to provide capacity hints, particularly where more than one blinded path is included, for payers to use multi-part payments.
Invoice Errors

Informative errors can be returned in an onion message invoice_error field (via the onion reply_path) for either invoice_request or invoice.
TLV Fields for invoice_error

    tlv_stream: invoice_error
    types:
        type: 1 (erroneous_field)
        data:
            [tu64:tlv_fieldnum]
        type: 3 (suggested_value)
        data:
            [...*byte:value]
        type: 5 (error)
        data:
            [...*utf8:msg]

Requirements

A writer of an invoice_error:

    MUST set error to an explanatory string.
    MAY set erroneous_field to a specific field number in the invoice or invoice_request which had a problem.
    if it sets erroneous_field:
        MAY set suggested_value.
        if it sets suggested_value:
            MUST set suggested_value to a valid field for that tlv_fieldnum.
    otherwise:
        MUST NOT set suggested_value.

A reader of an invoice_error: FIXME!
Rationale

Usually an error message is sufficient for diagnostics, however there is at least one case where it should be programatically parsable. A recurring offer which sets send_invoice can also specify a currency, which opens the possibility for a disagreement on exchange rate. In this case, the suggested_value reflects its expected value, and the sender can send a new invoice.
FIXME: Possible future extensions:

    The offer can require delivery info in the invoice_request.
    An offer can be updated: the response to an invoice_request is another offer, perhaps with a signature from the original node_id
    Any empty TLV fields can mean the value is supposed to be known by other means (i.e. transport-specific), but is still hashed for sig.
    We could upgrade to allow multiple offers in one invoice_request and invoice, to make a shopping list.
    All-zero offer_id == gratuitous payment.
    Streaming invoices?

[1] https://www.youtube.com/watch?v=4SYc_flMnMQ

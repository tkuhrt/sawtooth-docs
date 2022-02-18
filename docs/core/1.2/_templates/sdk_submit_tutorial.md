# "Client: Building and Submitting Transactions"

The process of encoding information to be submitted to a distributed
ledger is generally non-trivial. A series of cryptographic safeguards
are used to confirm identity and data validity. Hyperledger Sawtooth is
no different, but the {{ language }} SDK does provide client
functionality that abstracts away most of these details, and greatly
simplifies the process of making changes to the blockchain.

# Creating a Private Key and Signer

<!--
  Licensed under Creative Commons Attribution 4.0 International License
  https://creativecommons.org/licenses/by/4.0/
-->

In order to confirm your identity and sign the information you send to
the validator, you will need a 256-bit key. Sawtooth uses the secp256k1
ECDSA standard for signing, which means that almost any set of 32 bytes
is a valid key. It is fairly simple to generate a valid key using the
SDK\'s *signing* module.

A *Signer* wraps a private key and provides some convenient methods for
signing bytes and getting the private key\'s associated public key.

{% if language == \'JavaScript\' %}

``` javascript
const {createContext, CryptoFactory} = require('sawtooth-sdk/signing')

const context = createContext('secp256k1')
const privateKey = context.newRandomPrivateKey()
const signer = CryptoFactory(context).newSigner(privateKey)
```

{% elif language == \'Go\' %}

``` go
import "github.com/hyperledger/sawtooth-sdk-go/signing"

context := signing.NewSecp256k1Context()
privateKey := context.NewRandomPrivateKey()
signer := signing.NewCryptoFactory(context).NewSigner(privateKey)
```

{% elif language == \'Rust\' %}

``` rust
use sawtooth_sdk::signing::CryptoFactory;
use sawtooth_sdk::signing::create_context;

let context = create_context("secp256k1")
    .expect("Error creating the right context");
let private_key = context
    .new_random_private_key()
    .expect("Error generating a new Private Key");
let crypto_factory = CryptoFactory::new(context.as_ref());
let signer = crypto_factory.new_signer(private_key.as_ref());
```

{% else %}

{# Python 3 code should be the default #}

``` python
from sawtooth_signing import create_context
from sawtooth_signing import CryptoFactory

context = create_context('secp256k1')
private_key = context.new_random_private_key()
signer = CryptoFactory(context).new_signer(private_key)
```

{% endif %}

::: note
::: title
Note
:::

This key is the **only** way to prove your identity on the blockchain.
Any person possessing it will be able to sign Transactions using your
identity, and there is no way to recover it if lost. It is very
important that any private key is kept secret and secure.
:::

{% include \'partials/encoding_your_payload.rst\' %}

# Building the Transaction

*Transactions* are the basis for individual changes of state to the
Sawtooth blockchain. They are composed of a binary payload, a
binary-encoded *TransactionHeader* with some cryptographic safeguards
and metadata about how it should be handled, and a signature of that
header. It would be worthwhile to familiarize yourself with the
information in
`/architecture/transactions_and_batches`{.interpreted-text role="doc"},
particularly the definition of TransactionHeaders.

## 1. Create the Transaction Header

A TransactionHeader contains information for routing a transaction to
the correct transaction processor, what input and output state addresses
are involved, references to prior transactions it depends on, and the
public keys associated with the its signature. The header references the
payload through a SHA-512 hash of the payload bytes.

{% if language == \'JavaScript\' %}

``` javascript
const {createHash} = require('crypto')
const {protobuf} = require('sawtooth-sdk')

const transactionHeaderBytes = protobuf.TransactionHeader.encode({
    familyName: 'intkey',
    familyVersion: '1.0',
    inputs: ['1cf1266e282c41be5e4254d8820772c5518a2c5a8c0c7f7eda19594a7eb539453e1ed7'],
    outputs: ['1cf1266e282c41be5e4254d8820772c5518a2c5a8c0c7f7eda19594a7eb539453e1ed7'],
    signerPublicKey: signer.getPublicKey().asHex(),
    // In this example, we're signing the batch with the same private key,
    // but the batch can be signed by another party, in which case, the
    // public key will need to be associated with that key.
    batcherPublicKey: signer.getPublicKey().asHex(),
    // In this example, there are no dependencies.  This list should include
    // an previous transaction header signatures that must be applied for
    // this transaction to successfully commit.
    // For example,
    // dependencies: ['540a6803971d1880ec73a96cb97815a95d374cbad5d865925e5aa0432fcf1931539afe10310c122c5eaae15df61236079abbf4f258889359c4d175516934484a'],
    dependencies: [],
    payloadSha512: createHash('sha512').update(payloadBytes).digest('hex')
}).finish()
```

{% elif language == \'Go\' %}

``` go
import (
    "crypto/sha512"
    "encoding/hex"
    "github.com/golang/protobuf/proto"
    "github.com/hyperledger/sawtooth-sdk-go/protobuf/transaction_pb2"
    "strings"
)

hashHandler := sha512.New()
hashHandler.Write(payloadBytes)
payloadSha512 := strings.ToLower(hex.EncodeToString(hashHandler.Sum(nil)))

rawTransactionHeader := transaction_pb2.TransactionHeader{
    SignerPublicKey:  signer.GetPublicKey().AsHex(),
    FamilyName:       "intkey",
    FamilyVersion:    "1.0",
    // In this example, there are no dependencies.  This list should include
    // an previous transaction header signatures that must be applied for
    // this transaction to successfully commit.
    // For example,
    // dependencies:[]string{"540a6803971d1880ec73a96cb97815a95d374cbad5d865925e5aa0432fcf1931539afe10310c122c5eaae15df61236079abbf4f258889359c4d175516934484a"}
    Dependencies:     []string{},
    // In this example, we're signing the batch with the same private key,
    // but the batch can be signed by another party, in which case, the
    // public key will need to be associated with that key.
    BatcherPublicKey: signer.GetPublicKey().AsHex(),
    Inputs:           []string{"1cf1266e282c41be5e4254d8820772c5518a2c5a8c0c7f7eda19594a7eb539453e1ed7"},
    Outputs:          []string{"1cf1266e282c41be5e4254d8820772c5518a2c5a8c0c7f7eda19594a7eb539453e1ed7"},
    PayloadSha512:    payloadSha512,
}

// Check if err is nil before continuing
transactionHeaderBytes, err := proto.Marshal(&rawTransactionHeader)
```

{% elif language == \'Rust\' %}

``` rust
extern crate protobuf;
extern crate openssl;
extern crate rand;

use rand::{thread_rng, Rng};

use protobuf::Message
use protobuf::RepeatedField;

use openssl::sha::sha512;
use sawtooth_sdk::messages::transaction::TransactionHeader;

let mut txn_header = TransactionHeader::new();
txn_header.set_family_name(String::from("intkey"));
txn_header.set_family_version(String::from("1.0"));

// Generate a random 128 bit number to use as a nonce
let mut nonce = [0u8; 16];
thread_rng()
    .try_fill(&mut nonce[..])
    .expect("Error generating random nonce");
txn_header.set_nonce(to_hex_string(&nonce.to_vec()));

let input_vec: Vec<String> = vec![String::from(
    "1cf1266e282c41be5e4254d8820772c5518a2c5a8c0c7f7eda19594a7eb539453e1ed7",
)];
let output_vec: Vec<String> = vec![String::from(
    "1cf1266e282c41be5e4254d8820772c5518a2c5a8c0c7f7eda19594a7eb539453e1ed7",
)];

txn_header.set_inputs(RepeatedField::from_vec(input_vec));
txn_header.set_outputs(RepeatedField::from_vec(output_vec));
txn_header.set_signer_public_key(
    signer
        .get_public_key()
        .expect("Error retrieving Public Key")
        .as_hex(),
);
txn_header.set_batcher_public_key(
    signer
        .get_public_key()
        .expect("Error retrieving Public Key")
        .as_hex(),
);

txn_header.set_payload_sha512(to_hex_string(&sha512(&payload_bytes).to_vec()));

let txn_header_bytes = txn_header
    .write_to_bytes()
    .expect("Error converting transaction header to bytes");

// --snip--

// To properly format the Sha512 String
pub fn to_hex_string(bytes: &Vec<u8>) -> String {
    let strs: Vec<String> = bytes.iter()
        .map(|b| format!("{:02x}", b))
        .collect();
    strs.join("")
}
```

{% else %}

``` python
from hashlib import sha512
from sawtooth_sdk.protobuf.transaction_pb2 import TransactionHeader

txn_header_bytes = TransactionHeader(
    family_name='intkey',
    family_version='1.0',
    inputs=['1cf1266e282c41be5e4254d8820772c5518a2c5a8c0c7f7eda19594a7eb539453e1ed7'],
    outputs=['1cf1266e282c41be5e4254d8820772c5518a2c5a8c0c7f7eda19594a7eb539453e1ed7'],
    signer_public_key=signer.get_public_key().as_hex(),
    # In this example, we're signing the batch with the same private key,
    # but the batch can be signed by another party, in which case, the
    # public key will need to be associated with that key.
    batcher_public_key=signer.get_public_key().as_hex(),
    # In this example, there are no dependencies.  This list should include
    # an previous transaction header signatures that must be applied for
    # this transaction to successfully commit.
    # For example,
    # dependencies=['540a6803971d1880ec73a96cb97815a95d374cbad5d865925e5aa0432fcf1931539afe10310c122c5eaae15df61236079abbf4f258889359c4d175516934484a'],
    dependencies=[],
    payload_sha512=sha512(payload_bytes).hexdigest()
).SerializeToString()
```

{% endif %}

::: note
::: title
Note
:::

Remember that a *batcher public_key* is the hex public key matching the
private key that will later be used to sign a Transaction\'s Batch, and
*dependencies* are the *header signatures* of Transactions that must be
committed before this one (see *TransactionHeaders* in
`/architecture/transactions_and_batches`{.interpreted-text role="doc"}).
:::

::: note
::: title
Note
:::

The *inputs* and *outputs* are the state addresses a Transaction is
allowed to read from or write to. With the Transaction above, we
referenced the specific address where the value of `'foo'` is stored.
Whenever possible, specific addresses should be used, as this will allow
the validator to schedule transaction processing more efficiently.

Note that the methods for assigning and validating addresses are
entirely up to the Transaction Processor. In the case of IntegerKey,
there are [specific rules to generate valid addresses
\<../transaction_family_specifications
/integerkey_transaction_family.html#addressing>](), which must be
followed or Transactions will be rejected. You will need to follow the
addressing rules for whichever Transaction Family you are working with.
:::

## 2. Create the Transaction

Once the TransactionHeader is constructed, its bytes are then used to
create a signature. This header signature also acts as the ID of the
transaction. The header bytes, the header signature, and the payload
bytes are all used to construct the complete Transaction.

{% if language == \'JavaScript\' %}

``` javascript
const signature = signer.sign(transactionHeaderBytes)

const transaction = protobuf.Transaction.create({
    header: transactionHeaderBytes,
    headerSignature: signature,
    payload: payloadBytes
})
```

{% elif language == \'Go\' %}

``` go
import (
    "encoding/hex"
    "github.com/hyperledger/sawtooth-sdk-go/protobuf/transaction_pb2"
)

signature := hex.EncodeToString(signer.Sign(transactionHeaderBytes))
transaction := transaction_pb2.Transaction{
    Header:          transactionHeaderBytes,
    HeaderSignature: signature,
    Payload:         payloadBytes,
}
```

{% elif language == \'Rust\' %}

``` rust
use sawtooth_sdk::messages::transaction::Transaction;

let signature = signer
    .sign(&txn_header_bytes)
    .expect("Error signing the transaction header");

let mut txn = Transaction::new();
txn.set_header(txn_header_bytes.to_vec());
txn.set_header_signature(signature);
txn.set_payload(payload_bytes);
```

{% else %}

``` python
from sawtooth_sdk.protobuf.transaction_pb2 import Transaction

signature = signer.sign(txn_header_bytes)

txn = Transaction(
    header=txn_header_bytes,
    header_signature=signature,
    payload=payload_bytes
)
```

{% endif %}

## 3. (optional) Encode the Transaction(s)

If the same machine is creating Transactions and Batches there is no
need to encode the Transaction instances. However, in the use case where
Transactions are being batched externally, they must be serialized
before being transmitted to the batcher. The {{ language }} SDK offers
two options for this. One or more Transactions can be combined into a
serialized *TransactionList* method, or can be serialized as a single
Transaction.

{% if language == \'JavaScript\' %}

``` javascript
const txnListBytes = protobuf.TransactionList.encode([
    transaction1,
    transaction2
]).finish()

const txnBytes2 = transaction.finish()
```

{% elif language == \'Go\' %}

``` go
import (
    "github.com/golang/protobuf/proto"
    "github.com/hyperledger/sawtooth-sdk-go/protobuf/transaction_pb2"
)

rawTransactionList := transaction_pb2.TransactionList{
    Transactions: []*transaction_pb2.Transaction{
        &transaction1,
        &transaction2,
    },
}
// Check if err is nil before continuing
transactionListBytes, err := proto.Marshal(&rawTransactionList)

// Check if err is nil before continuing
transactionBytes, err := proto.Marshal(&transaction)
```

{% elif language == \'Rust\' %}

``` rust
let txn_list_vec = vec![txn1, txn2];
let txn_list = TransactionList::new();
txn_list.set_transactions(RepeatedField::from_vec(txn_list_vec));

let txn_list_bytes = txn_list
    .write_to_bytes()
    .expect("Error converting Transaction List to bytes");
```

{% else %}

``` python
from sawtooth_sdk.protobuf.transaction_pb2 import TransactionList

txn_list_bytes = TransactionList(
    transactions=[txn1, txn2]
).SerializeToString()

txn_bytes = txn.SerializeToString()
```

{% endif %}

# Building the Batch

Once you have one or more Transaction instances ready, they must be
wrapped in a *Batch*. Batches are the atomic unit of change in
Sawtooth\'s state. When a Batch is submitted to a validator each
Transaction in it will be applied (in order), or *no* Transactions will
be applied. Even if your Transactions are not dependent on any others,
they cannot be submitted directly to the validator. They must all be
wrapped in a Batch.

## 1. Create the BatchHeader

Similar to the TransactionHeader, there is a *BatchHeader* for each
Batch. As Batches are much simpler than Transactions, a BatchHeader
needs only the public key of the signer and the list of Transaction IDs,
in the same order they are listed in the Batch.

{% if language == \'JavaScript\' %}

``` javascript
const transactions = [transaction]

const batchHeaderBytes = protobuf.BatchHeader.encode({
    signerPublicKey: signer.getPublicKey().asHex(),
    transactionIds: transactions.map((txn) => txn.headerSignature),
}).finish()
```

{% elif language == \'Go\' %}

``` go
import (
    "github.com/golang/protobuf/proto"
    "github.com/hyperledger/sawtooth-sdk-go/protobuf/batch_pb2"
)

transactionSignatures := []string{transaction.HeaderSignature}

rawBatchHeader := batch_pb2.BatchHeader{
    SignerPublicKey: signer.GetPublicKey().AsHex(),
    TransactionIds:  transactionSignatures,
}

// Check if err is nil before continuing
batchHeaderBytes, err := proto.Marshal(&rawBatchHeader)
```

{% elif language == \'Rust\' %}

``` rust
use sawtooth_sdk::messages::batch::BatchHeader;

let mut batch_header = BatchHeader::new();

batch_header.set_signer_public_key(
    signer
        .get_public_key()
        .expect("Error retrieving Public Key")
        .as_hex(),
);

let transaction_ids = vec![txn.clone()]
    .iter()
    .map(|trans| String::from(trans.get_header_signature()))
    .collect();

batch_header.set_transaction_ids(RepeatedField::from_vec(transaction_ids));

let batch_header_bytes = batch_header
    .write_to_bytes()
    .expect("Error converting batch header to bytes");
```

{% else %}

``` python
from sawtooth_sdk.protobuf.batch_pb2 import BatchHeader

txns = [txn]

batch_header_bytes = BatchHeader(
    signer_public_key=signer.get_public_key().as_hex(),
    transaction_ids=[txn.header_signature for txn in txns],
).SerializeToString()
```

{% endif %}

## 2. Create the Batch

Using the SDK, creating a Batch is similar to creating a transaction.
The header is signed, and the resulting signature acts as the Batch\'s
ID. The Batch is then constructed out of the header bytes, the header
signature, and the transactions that make up the batch.

{% if language == \'JavaScript\' %}

``` javascript
const signature = signer.sign(batchHeaderBytes)

const batch = protobuf.Batch.create({
    header: batchHeaderBytes,
    headerSignature: signature,
    transactions: transactions
})
```

{% elif language == \'Go\' %}

``` go
import (
    "encoding/hex"
    "github.com/hyperledger/sawtooth-sdk-go/protobuf/batch_pb2"
)

signature := hex.EncodeToString(signer.Sign(batchHeader))

batch := batch_pb2.Batch{
    Header:          batchHeaderBytes,
    Transactions:    transactions,
    HeaderSignature: signature,
}
```

{% elif language == \'Rust\' %}

``` rust
use sawtooth_sdk::messages::batch::Batch;

let signature = signer
    .sign(&batch_header_bytes)
    .expect("Error signing the batch header");

let mut batch = Batch::new();

batch.set_header(batch_header_bytes);
batch.set_header_signature(signature);
batch.set_transactions(RepeatedField::from_vec(vec![txn]));
```

{% else %}

``` python
from sawtooth_sdk.protobuf.batch_pb2 import Batch

signature = signer.sign(batch_header_bytes)

batch = Batch(
    header=batch_header_bytes,
    header_signature=signature,
    transactions=txns
)
```

{% endif %}

## 3. Encode the Batch(es) in a BatchList

In order to submit Batches to the validator, they must be collected into
a *BatchList*. Multiple batches can be submitted in one BatchList,
though the Batches themselves don\'t necessarily need to depend on each
other. Unlike Batches, a BatchList is not atomic. Batches from other
clients may be interleaved with yours.

{% if language == \'JavaScript\' %}

``` javascript
const batchListBytes = protobuf.BatchList.encode({
    batches: [batch]
}).finish()
```

{% elif language == \'Go\' %}

``` go
import (
    "github.com/golang/protobuf/proto"
    "github.com/hyperledger/sawtooth-sdk-go/protobuf/batch_pb2"
)

rawBatchList := batch_pb2.BatchList{
    Batches: []*batch_pb2.Batch{&batch},
}

// Check if err is nil before continuing
batchListBytes := proto.Marshal(&rawBatchList)
```

{% elif language == \'Rust\' %}

``` rust
use sawtooth_sdk::messages::batch::BatchList;

let mut batch_list = BatchList::new();
batch_list.set_batches(RepeatedField::from_vec(vec![batch]));
let batch_list_bytes = batch_list
    .write_to_bytes()
    .expect("Error converting batch list to bytes");
```

{% else %}

``` python
from sawtooth_sdk.protobuf.batch_pb2 import BatchList

batch_list_bytes = BatchList(batches=[batch]).SerializeToString()
```

{% endif %}

::: note
::: title
Note
:::

Note, if the transaction creator is using a different private key than
the batcher, the *batcher public_key* must have been specified for every
Transaction, and must have been generated from the private key being
used to sign the Batch, or validation will fail.
:::

{% include \'partials/submitting_to_validator.rst\' %}
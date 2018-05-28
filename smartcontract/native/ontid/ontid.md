English / [中文](./ontid_cn.md)

<h1 align="center">Ontology Distributed Identification</h1>
<p align="center" class="version">Contract Version 1.0</p>

“Entity” refers to individuals, legal entities (organizations, enterprises, institutions, etc.), objects (mobile phones, automobiles, IoT devices, etc.), and contents (articles, copyrights, etc.) in the real world, and “identity” refers to the entity's identity within the network. Ontology uses Ontology Identifier (ONT ID) to identify and manage the entities' identities. On Ontology Blockchain, one entity can correspond to multiple individual identities, and there is no relation between multiple identities.

The ONT ID is a decentralized identification protocol and it has the features of decentralization, self-management, privacy protection, security and ease of use. Each ONT ID corresponds to an ONT ID Description Object (DDO).

> The ONT ID protocol has been completely implemented by the smart contract of Ontology Blockchain. As a protocol layer, it follows a decoupled design, so it is not limited to Ontology Blockchain, but can also be implemented on other blockchains.

## 1. Identification Protocol Specification

### 1.1 ONT ID generation

The ONT ID is a URI that is generated by each entity itself. The generation algorithm needs to guarantee that the collision probability is extremely low. Beside, when someone register an ONT ID on Ontology, the consensus node can check whether the ID is already registered.

ONT ID generation algorithm:

To prevent the user from entering the ONT ID by mistake, we define a valid ONT ID that must contain 4 bytes of verification data. We are going to describe in detail how to generate a valid ONT ID.
 1. Generate a 32-byte temporary random nonce, and calculate h = Hash160 (nonce), data = `<VER>` || h;
 2. Calculate a 4-byte verification data, that is, checksum = SHA256(SHA256(data))[0:3];
 3. Make idString = base58(data || checksum);
 4. Concat "did:`<ont>`:" with idString, that is, ontId = "did:`<ont>`:" || idString;
 5. Output ONT ID.

As above, `<ont>` is a network identifier, and `<VER>` is a 1 byte version label. In ONT, `<VER> = 41, <ont> = "ont"`. That is to say , the first 8 bytes of identity in Ontology are "did:ont:", plus a 25 byte long idString, which constitutes a complete ONT ID.

### 1.2 Self-management
Ontology applies digital signature technology to guarantee entities have rights to manage their own identities. The ONT ID is bound to the entity's public key when it registers, thereby indicating its ownership. The use of the ONT ID and the modification of its attributes require the owner's digital signature. The entity can independently determine the scope of use of its ONT ID and set the public key bounded by ONT ID and manage the attributes of the ONT ID.

### 1.3 Multiple key binding
Ontology supports a variety of domestic and international standardized digital signature algorithms such as ECDSA and SM2. The algorithm applied to the key that is bounded by ONT ID should be specified. At the same time, an ONT ID can bound multiple different keys to meet the usage requirements of entities in different application scenarios.

### 1.4 Recovery of identity loss
The owner of the ONT ID can assign someone else to execute his management rights, such as modifying the attributes of the ONT ID and replacing the key when the key is lost. The assigned person can implement a variety of access control logic such as “AND”, “OR”, and “(m, n)-thresholds”. Refer to [Appendix B](#b.-recovery-account-address) for more details.

### 1.5 Identity description object DDO specification

The identity description object DDO corresponding to the ONT ID is stored on the Ontology Blockchain. It is written to the blockchain by the controller of the DDO and is open to all users for reading.

The DDO specification contains the following information:
- `PublicKeys`：The information of the public key used for identity authentication, including public key id, public key type, and public key data;
- `Attributes`：All attributes make up a JSON object;
- `Recovery`：The assigned restorer can help reset the user's public key list.

For an example in json format,

```json
{
	"OntId": "did:ont:TVuF6FH1PskzWJAFhWAFg17NSitMDEBNoa",
	"Owners": [{
			"PubKeyId": "did:ont:TVuF6FH1PskzWJAFhWAFg17NSitMDEBNoa#keys-1",
			"Type": "ECDSA",
			"Curve": "nistp256",
			"Value":"022f71daef10803ece19f96b2cdb348d22bf7871c178b41f35a4f3772a8359b7d2"
		}, {
			"PublicKeyId": "did:ont:TVuF6FH1PskzWJAFhWAFg17NSitMDEBNoa#keys-2", 
			"Type": "RSA",
			"Length": 2048, 
			"Value": "3082010a...."
		}
	],
	"Attributes": {
		"OfficialCredential": {
			"Service": "PKI",
			"CN": "ont.io",
			"CertFingerprint": "1028e8f7043f12c0c2069bd7c7b3b26213964566"
		}
	},
	"Recovery": "TA63T1gxXPXWsBqHtBKcV4NhFBhw3rtkAF"
}
```

## 2. Smart Contract Implementation Specification

ONT ID protocol is implemented as a native contract on the Ontology Blockchain platform. With the ONT ID contract, users can register IDs, manage their own public key lists, modify their personal profiles, and add account restorers.

Every non-query methods will push an event message to notify the caller when it executes successfully. Please refer to the [Events](#2.2.-events) subsection.

## 2.1 Methods

**Note**: All the user's public key used as arguments of following methods should be the transaction constructor's public key, i.e. it can pass the transaction's signature verification. And the public key should be bound to the user's ID except for the register methods.

**Note1**: Please refer to the [contract development specification](./development_specification_cn.md) for the argument serialization rules.

###  a. Identity registration

When a user registers a ID, he must submit a public key as the initial key.

```
Method: regIDWithPublicKey

Arguments:
    0    String      user ID
    1    ByteArray   public key

Event: triger 'Register' event when succeeds
```
 
###  b. Add a control key

The user adds a new public key to his public key list.

```
Method: addKey

Arguments:
    0    String       user ID
    1    ByteArray    new public key to be added
    2    ByteArray    user's public key or the recovery address

Event: trigger 'PublicKey' event when succeeds
```

### c. Delete a control key

Remove a public key from the user's public key list.

```
Method: removeKey

Arguments:
    0    String       user ID
    1    ByteArray    public key to be removed
    2    ByteArray    user's public key or the recovery address

Event: trigger 'PublicKey' event when succeeds
```
	
### d. Key recovery mechanism

Add or modify the recovery address.

```
Method: addRecovery

Arguments:
    0    String       user ID
    1    Address      recovery address
    2    ByteArray    user's public key

Event: trigger 'Recovery' event when succeeds
```

This method secceeds if and only if the argument 2 is the user's existing public key,
and the recovery address has not been set.


```
Method: changeRecovery

Arguments:
    0    String     user ID
    1    Address    new recovery address
    2    Address    original recovery address

Event: trigger 'Recovery' event when succeeds
```

This contract call must be initiated by the original recovery address.

### e. Attribute management

An attribute consists of the following 3 parameters:

```
Attribute {
    ByteArray path
    ByteArray type
    ByteArray value
}
```

While register an ID, user can set attributes at the same time using method `regIDWithAttributes`.

```
Method: regIDWithAttributes

Arguments:
    0    String             user ID
    1    ByteArray          user's public key
    2    Attribute Array    attributes
```

The addition, deletion, and modification of the user’s attributes must be authorized by the user.

```
Method: addAttributes

Arguments:
    0    String             user ID
    1    Attribute Array    attributes
    2    ByteArray          user's public key

Event: trigger 'Attribute' event when succeeds
```

If an attribute does not exist, the attribute will be added. Otherwise the
original attribute will be updated.

```
Method: removeAttribute

Arguments:
    0    String       user ID
    1    ByteArray    path of the attribute to be deleted
    2    ByteArray    user's public key

Event: trigger 'Attribute' event when succeeds
```


### f. Query identity information

**Keys**

```
Method: getPublicKeys

Argument: String, which is user's ID

Return: ByteArray
```

Each public key contains following attributes:

```
publicKey {
    Int      index
    ByteArray data
}
```

And the returned byte array consists of a sequence of serialized public keys,
i.e. serialization of `publicKey[]`.

The index is the number appears in the `PubKeyID` of DDO. It is generated
automatically when registered or added, increasingly. Index of revoked key will
not be recycled. Instead the key state will be changed to 'revoked'. Use the
following method to get key state:

```
Method: getKeyState

Argument:
    0    String  user's ID
    1    Int    key index

Return: "in use" | "revoked" | "not exist"
```

**Attributes**

```
Method: getAttributes

Argument: String, which is user's ID

Return: ByteArray
```

The returned byte array consists of a sequence of serialized attributes, whose 
structure is defined as the above.

**DDO**

```
Method: getDDO

Argument: String, which is user's ID

Return: ByteArray 
```

The returned value contains the result of `GetPublicKeys` and `GetAttributes`,
as well as the recovery address:

```
ddo {
    ByteArray keys         // serialization of publicKey[]
    ByteArray attributes   // serialization of attribute[]
    ByteArray recovery     // recovery address
}
```

### g. Verify signature

```
Method: verifySignature

Argument:
    0    String    user's ID
    1    Int      key index

Return: true | false
```

This method is mainly used by other contracts for authentication. It checks
whether the contract is invoked by the specified key.

## 2.2. Events

There are three kinds of event messages:

- `Register`:  Push the messages related to identity registration.

	| Field | Type       | Description       |
	| :---- | :--------- | :---------------- |
	|  op   | String     | message type      |
	|  ID   | ByteArray | registered ONT ID |

- `PublicKey`: Push the messages related to public key operations.

	| Field      | Type       | Description    |
	| :--------- | :--------- | :--------------|
	|  op        | String     | message type："add" or "remove" |
	|  ID        | ByteArray  | user's ONT ID    |
	| key index  | Int       | index of the key |
	| public key | ByteArray  | public key data  |

- `Attribute`: Push the messages related to attribute operations.

	| Field    | Type       | Description    |
	| :------- | :--------- | :------------- | 
	| op       | String     | message type："add"、remove"  |
	| ID       | ByteArray | user's ONT ID|
	| attrName | ByteArray | attribute name |
	
- `Recovery`: Push the messages related to recovery operations.

	| Field    | Type       | Description    |
	| :------- | :--------- | :------------- | 
	| op       | String     | message type："add"、"change" |
	| ID       | ByteArray | user's ONT ID |
	| address  | ByteArray | recovery address |

## Appendix

### A. Public key data

The public key data of type byte array follows [ontology-crypto serialization](https://github.com/ontio/ontology-crypto/wiki/Asymmetric-Keys#Serialization) format:

    algorithm_id || algorithm_parameters || public_key


### B. Recovery account address

The recovery account can implement a variety of access control logic, such as (m,n)-threshold control. A (m,n) threshold control account is managed by n public keys altogether. To use it, you have to gather at least m valid signatures.

- `(m, n) threshold` control account

	```
	0x02 || RIPEMD160(SHA256(n || m || publicKey_1 || ... || publicKey_n))
	```

- `AND` control account
   
   This is equivalent to (n, n) threshold control account.

- `OR` control account
  
   This is equivalent to (1, n) threshold control account.

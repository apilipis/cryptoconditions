[![Build Status](https://travis-ci.org/bigchaindb/cryptoconditions.svg?branch=master)](https://travis-ci.org/bigchaindb/cryptoconditions)
[![codecov.io](https://codecov.io/github/bigchaindb/cryptoconditions/coverage.svg?branch=master)](https://codecov.io/github/bigchaindb/cryptoconditions?branch=master)


# How to install and run tests

First clone this repository (optional: and a virtual env).
Note that we support **Python>=3.4**.

```
$ pip install -e .[dev]
$ py.test -v
```


# Crypto Conditions

This spec is a python port from the [**Interledger Protocol (ILP)**]
(https://interledger.org/five-bells-condition/spec.html)

## Motivation

We would like a way to describe a signed message such that multiple actors in a
distributed system can all verify the same signed message and agree on whether
it matches the description.

This provides a useful primitive for distributed, event-based systems since we
can describe events (represented by signed messages) and therefore define
generic authenticated event handlers.

## Usage

```python
import binascii
from cryptoconditions.condition import Condition
from cryptoconditions.fulfillment import Fulfillment
from cryptoconditions.fulfillments.sha256 import PreimageSha256Fulfillment

# Parse a condition from a URI
example_condition_uri = 'cc:1:1:47DEQpj8HBSa-_TImW-5JCeuQeRkm5NMpJWZG3hSuFU:1'
parsed_condition = Condition.from_uri(example_condition_uri)
print(isinstance(parsed_condition, Condition))
# prints True

print(binascii.hexlify(parsed_condition.hash))
# prints b'e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855'

# Compile a condition
parsed_condition_uri = parsed_condition.serialize_uri()
print(parsed_condition_uri)
# prints 'cc:1:3:47DEQpj8HBSa-_TImW-5JCeuQeRkm5NMpJWZG3hSuFU:1'
print(parsed_condition_uri == example_condition_uri)
# prints True

# Parse a fulfillment
example_fulfillment_uri = 'cf:1:0:AA'
parsed_fulfillment = Fulfillment.from_uri(example_fulfillment_uri)
print(isinstance(parsed_fulfillment, PreimageSha256Fulfillment))
# prints True
# Note: Merely parsing a fulfillment DOES NOT validate it.

# Validate a fulfillment
parsed_fulfillment.validate()
# prints True
```

## ILP Format

### Condition

Conditions are ASCII encoded as:

```
"cc:" BASE10(VERSION) ":" BASE16(TYPE_BITMASK) ":" BASE64URL(HASH) ":" BASE10(FULFILLMENT_LENGTH)
```

Conditions are binary encoded as:

```
CONDITION =
  VARUINT TYPE_BITMASK
  VARBYTES HASH
  VARUINT FULFILLMENT_LENGTH
```

### Fulfillment

Fulfillments are ASCII encoded as:

```
"cf:" BASE10(VERSION) ":" BASE16(TYPE_BIT) ":" BASE64URL(FULFILLMENT_PAYLOAD)
```

Fulfillments are binary encoded as:

```
FULFILLMENT =
  VARUINT TYPE_BIT
  FULFILLMENT_PAYLOAD
```

# Condition Types

## SHA-256

### Condition

```
HASH = SHA256(PREIMAGE)
```

### Fulfillment

```
FULFILLMENT_PAYLOAD =
  VARBYTES PREIMAGE
```

### Usage

```python
import binascii, hashlib
from cryptoconditions.condition import Condition
from cryptoconditions.fulfillments.sha256 import PreimageSha256Fulfillment

secret = ''
puzzle = binascii.hexlify(hashlib.sha256(secret.encode()).digest())
print(puzzle)
# prints b'e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855'

# Create a SHA256 condition
sha256condition = Condition()
sha256condition.bitmask = PreimageSha256Fulfillment.FEATURE_BITMASK
sha256condition.hash = binascii.unhexlify(puzzle)
sha256condition.max_fulfillment_length = 1
sha256condition_uri = sha256condition.serialize_uri()
print(sha256condition_uri)
# prints 'cc:1:3:47DEQpj8HBSa-_TImW-5JCeuQeRkm5NMpJWZG3hSuFU:1'

# Create a fulfillment
sha256fulfillment = PreimageSha256Fulfillment()

# Create a condition from fulfillment
sha256fulfillment.condition
# raises ValueError: Could not calculate hash, no preimage provided
sha256fulfillment.preimage = secret
print(sha256fulfillment.condition.serialize_uri() == sha256condition_uri)
# prints True

# Compile a fulfillment
print(sha256fulfillment.serialize_uri())
# prints 'cf:1:0:AA'

# Even better: verify that the fulfillment matches the condition
print(sha256fulfillment.validate() and \
    sha256fulfillment.condition.serialize_uri() == sha256condition.serialize_uri())
# prints True
```

## RSA-SHA-256

**Warning:** not (yet) implemented in `cryptoconditions`, for info see the
[**ILP specification**](https://interledger.org/five-bells-condition/spec.html)

## ED25519

### Condition

```
HASH = UINT256 PUBLIC_KEY
```

### Fulfillment

```
FULFILLMENT_PAYLOAD =
  UINT256 PUBLIC_KEY
  UINT512 SIGNATURE
```

### Usage

```python
from cryptoconditions.ed25519 import Ed25519SigningKey, Ed25519VerifyingKey
from cryptoconditions.fulfillments.ed25519 import Ed25519Fulfillment

# We use base58 key encoding
sk = Ed25519SigningKey(b'9qLvREC54mhKYivr88VpckyVWdAFmifJpGjbvV5AiTRs')
vk = sk.get_verifying_key()

# Create an ED25519-SHA256 condition
ed25519_fulfillment = Ed25519Fulfillment(public_key=vk)
ed25519_condition_uri = ed25519_fulfillment.condition.serialize_uri()
print (ed25519_condition_uri)
# prints 'cc:1:20:7Bcrk61eVjv0kyxw4SRQNMNUZ-8u_U1k6_gZaDRn4r8:98'

# ED25519-SHA256 condition not fulfilled
print(ed25519_fulfillment.validate())
# prints False

# Fulfill an ED25519-SHA256 condition
message = 'Hello World! Conditions are here!'
ed25519_fulfillment.sign(message, sk)
print(ed25519_fulfillment.validate(message))
# prints True

print(ed25519_fulfillment.serialize_uri())
# prints
#  'cf:1:4:IOwXK5OtXlY79JMscOEkUDTDVGfvLv1NZOv4GWg0Z-K_QLYikfrZQy-PKYucSk'
#  'iV2-KT9v_aGmja3wzN719HoMchKl_qPNqXo_TAPqny6Kwc7IalHUUhJ6vboJ0bbzMcBwo'
print(ed25519_fulfillment.condition.serialize_uri())
# prints 'cc:1:20:7Bcrk61eVjv0kyxw4SRQNMNUZ-8u_U1k6_gZaDRn4r8:98'

# Parse a fulfillment URI
parsed_ed25519_fulfillment = Ed25519Fulfillment.from_uri('cf:1:4:IOwXK5OtXlY79JMscOEkUDTDVGfvLv1NZOv4GWg0Z-K_QLYikfrZQy-PKYucSkiV2-KT9v_aGmja3wzN719HoMchKl_qPNqXo_TAPqny6Kwc7IalHUUhJ6vboJ0bbzMcBwo')

print(parsed_ed25519_fulfillment.validate(message))
# prints True
print(parsed_ed25519_fulfillment.condition.serialize_uri())
# prints 'cc:1:20:7Bcrk61eVjv0kyxw4SRQNMNUZ-8u_U1k6_gZaDRn4r8:98'
```

## THRESHOLD-SHA-256

### Condition

```
HASH = SHA256(
  VARUINT TYPE_BIT
  VARUINT THRESHOLD
  VARARRAY
    VARUINT WEIGHT
    VARBYTES PREFIX
    CONDITION
)
```


### Fulfillment

```
FULFILLMENT_PAYLOAD =
  VARUINT THRESHOLD
  VARARRAY
    UINT8 FLAGS
    OPTIONAL VARUINT WEIGHT   ; if  FLAGS & 0x40
    OPTIONAL VARBYTES PREFIX  ; if  FLAGS & 0x20
    OPTIONAL FULFILLMENT      ; if  FLAGS & 0x80
    OPTIONAL CONDITION        ; if ~FLAGS & 0x80
```

### Usage

```python
from cryptoconditions.fulfillments.sha256 import PreimageSha256Fulfillment
from cryptoconditions.fulfillments.ed25519 import Ed25519Fulfillment
from cryptoconditions.fulfillments.threshold_sha256 import ThresholdSha256Fulfillment

# Parse some fulfillments
sha256_fulfillment = Sha256Fulfillment.from_uri('cf:1:0:AA')
ed25519_fulfillment = Ed25519Fulfillment.from_uri('cf:1:4:IOwXK5OtXlY79JMscOEkUDTDVGfvLv1NZOv4GWg0Z-K_QLYikfrZQy-PKYucSkiV2-KT9v_aGmja3wzN719HoMchKl_qPNqXo_TAPqny6Kwc7IalHUUhJ6vboJ0bbzMcBwo')

# Create a threshold condition
threshold_fulfillment = ThresholdSha256Fulfillment()
threshold_fulfillment.add_subfulfillment(sha256_fulfillment)
threshold_fulfillment.add_subfulfillment(ed25519_fulfillment)
threshold_fulfillment.threshold = 1  # OR gate
print(threshold_fulfillment.condition.serialize_uri())
# prints 'cc:1:29:ehSJGVIK3HVpRD0vEWHs9m7gez0q2Qm8C0DSK5bQ1zk:105'

# Compile a threshold fulfillment
threshold_fulfillment_uri = threshold_fulfillment.serialize_uri()
# Note: If there are more than enough fulfilled subconditions, shorter
# fulfillments will be chosen over longer ones.
print(threshold_fulfillment_uri)
# prints 'cf:1:2:AQEBYwQg7Bcrk61eVjv0kyxw4SRQNMNUZ-8u_U1k6_gZaDRn4r9AtiKR-tlDL48pi5xKSJXb4pP2_9oaaNrfDM3vX0egxyEqX-o82pej9MA-qfLorBzshqUdRSEnq9ugnRtvMxwHCgA'

# Validate fulfillment
message = 'Hello World! Conditions are here!'
print(threshold_fulfillment.validate(message))
# prints True

# Parse the fulfillment
reparsed_fulfillment = \
    ThresholdSha256Fulfillment.from_uri(threshold_fulfillment_uri)
print(reparsed_fulfillment.validate(message))
# prints True

# Increase threshold to a 3-port AND gate
threshold_fulfillment.threshold = 3
print(threshold_fulfillment.validate(message))
# prints False

# Create a nested threshold condition
# VALID = SHA and DSA and (DSA or DSA)
nested_fulfillment = ThresholdSha256Fulfillment()
nested_fulfillment.add_subfulfillment(ed25519_fulfillment)
nested_fulfillment.add_subfulfillment(ed25519_fulfillment)
nested_fulfillment.threshold = 1  # OR gate
threshold_fulfillment.threshold = 2 # AND gate

threshold_fulfillment.add_subfulfillment(nested_fulfillment)
print(threshold_fulfillment.serialize_uri())
# prints
#  'cf:1:2:AgIBYwQg7Bcrk61eVjv0kyxw4SRQNMNUZ-8u_U1k6_gZaDRn4r9AtiKR-tlDL'
#  '48pi5xKSJXb4pP2_9oaaNrfDM3vX0egxyEqX-o82pej9MA-qfLorBzshqUdRSEnq9ugn'
#  'RtvMxwHCgABjwECAQIBACMgIOwXK5OtXlY79JMscOEkUDTDVGfvLv1NZOv4GWg0Z-K_Y'
#  'gFjBCDsFyuTrV5WO_STLHDhJFA0w1Rn7y79TWTr-BloNGfiv0C2IpH62UMvjymLnEpIl'
#  'dvik_b_2hpo2t8Mze9fR6DHISpf6jzal6P0wD6p8uisHOyGpR1FISer26CdG28zHAcKAAA'
```

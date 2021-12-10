---
layout: default
title: following a function through the SDK - delegating to a pillar
parent: deep dives
grand_parent: dart SDK documentation
nav_order: 2
---

# Deep dive - follow a function through the SDK: delegating to a pillar

## Summary

The `delegate()` function in the pillarApi class generates an account block from the `accountBlockTemplate` and executes its `callContract` function to broadcast the transaction authorising delegation to the named pillar.

## Breakdown

Having imported the SDK:

```dart
    import 'package:znn_sdk_dart/znn_sdk_dart.dart';
```

We initialize it by creating a Zenon object:

```dart
    var zenon = Zenon();
```

And, ignoring the setup of the node connection (deep dive on that subject coming soon), we can use the zenon object to delegate to a pillar and get the result of our smart contract interaction:

```dart
    var delegated = zenon.embedded.pillar.delegate('pillar');
```

So, let's dive in to this a little bit. In `znn_sdk_dart.dart` we see that `Zenon` is imported from `zenon.dart`. In turn, in `zenon.dart` we can see in lines 26-29 that the API functions we call to interact with the network all come from the various API classes.

```dart
    late api.LedgerApi ledger;
    late api.StatsApi stats;
    late api.EmbeddedApi embedded;
    late api.SubscribeApi subscribe;
```

The `embeddedApi` class in `lib/src/api/embedded.dart` which allows interactions with the Zenon embedded smart contracts gives access to the pillar functions through the `pillarApi` class in `lib/src/api/embedded/pillar.dart`. There, in lines 113-116 we find our delegate function, which is where it starts to get interesting:

```dart
    AccountBlockTemplate delegate(String name) {
      return AccountBlockTemplate.callContract(
          pillarAddress, znnZts, 0, Definitions.pillar.encodeFunction('Delegate', [name]));
    }
```

`AccountBlockTemplate` is the class which generates "account blocks", the form a transaction takes in a block-lattice ledger. In this case, it will generate a contract-calling transaction which takes the following arguments:

```dart
factory AccountBlockTemplate.callContract(Address toAddress, TokenStandard tokenStandard, int amount, List<int> data)
```


Let's look at the arguments provided to the `AccountBlockTemplate.callContract()` function.

- toAddress: `pillarAddress` is imported from `lib/src/model/primitives/address.dart`, which stores a hardcoded list of addresses to which the embedded smart contract calls are made.

```dart
final Address pillarAddress =
    Address.parse('z1qxemdeddedxpyllarxxxxxxxxxxxxxxxsy3fmg');
```

Here we see the address `z1qxemdeddedxpyllarxxxxxxxxxxxxxxxsy3fmg` - this is the address of the embedded pillar smart contract where transactions relevant to pillars can be made.


- tokenStandard: `znnZts` is imported from `lib/src/model/primitives/token_standard.dart`. The relevant code is:

```dart
const String znnTokenStandard = 'zts1znnxxxxxxxxxxxxx9z4ulx';
...
final TokenStandard znnZts = TokenStandard.parse(znnTokenStandard);
...
class TokenStandard {
  static const String prefix = 'zts';
  static const int coreSize = 10;

  late String hrp;
  late List<int> core;

  TokenStandard.parse(String tokenStandard) {
    var bech32 = bech32Codec.decode(tokenStandard);
    hrp = bech32.hrp;
    core = convertBech32Bits(bech32.data, 5, 8, false);
    validate();
  }
...
```

In terms of how the network is structured, we can see that everything, even ZNN, follows zenon token standard (ZTS) logic. ZNN is simply a special, hardcoded token on the network of momentum. 

- amount: `0`. No ZNN is transacted in the delegation contract call.

- data: `Definitions.pillar.encodeFunction('Delegate', [name])`. In `src/embedded/definitions.dart`, the functions which embedded smart contracts can use are layed out. For pillars, it looks like this:

```dart
  static final String _pillarDefinition = '''[
    {"type":"function","name":"Register","inputs":[{"name":"name","type":"string"},{"name":"producerAddress","type":"address"},{"name":"rewardAddress","type":"address"},{"name":"giveBlockRewardPercentage","type":"uint8"},{"name":"giveDelegateRewardPercentage","type":"uint8"}]},
    {"type":"function","name":"RegisterLegacy","inputs":[{"name":"name","type":"string"},{"name":"producerAddress","type":"address"},{"name":"rewardAddress","type":"address"},{"name":"giveBlockRewardPercentage","type":"uint8"},{"name":"giveDelegateRewardPercentage","type":"uint8"},{"name":"publicKey","type":"string"},{"name":"signature","type":"string"}]},
    {"type":"function","name":"Revoke","inputs":[{"name":"name","type":"string"}]},
    {"type":"function","name":"UpdatePillar","inputs":[{"name":"name","type":"string"},{"name":"producerAddress","type":"address"},{"name":"rewardAddress","type":"address"},{"name":"giveBlockRewardPercentage","type":"uint8"},{"name":"giveDelegateRewardPercentage","type":"uint8"}]},
    {"type":"function","name":"Delegate","inputs":[{"name":"name","type":"string"}]},
    {"type":"function","name":"Undelegate","inputs":[]},
    {"type":"function","name":"UpdateEmbeddedPillar","inputs":[]}
  ]''';
```

We can see our delegate function here. Later on in `definitions.dart`, that list of functions has been converted into an ABI:

```dart
    static final Abi pillar = Abi.fromJson(_pillarDefinition);
```

This is where `encodeFunction` is being called from. It looks like this:

```dart
  List<int> encodeFunction(String name, var args) {
    AbiFunction? f;
    entries.forEach((element) {
      if (element.name == name) {
        f = AbiFunction(element.name!, element.inputs!);
      }
    });
    return f!.encode(args);
  }
```
and `encode()`:
```dart
  List<int> encode(dynamic args) {
    var l1 = encodeArguments(args);
    return BytesUtils.merge([encodeSignature(), l1]);
  }
```
Gives a merged array of the arguments and a [SHA-3](https://en.wikipedia.org/wiki/SHA-3) hash of the arguments formatted as a list of integers. 

(By the way, those exclamation marks are [null cast-away operators](https://dart.dev/null-safety/understanding-null-safety#null-assertion-operator) assert that those values will never be null at runtime, helping with compiler optimization)

## Putting it all back together

Putting it all back together, the contract call consists of:
- The pillar contract address
- The ZNN token standard
- 0 coins
- The "delegate function", the name of the pillar, and a hash signature as transaction "data".

This is then formatted as an otherwise normal "send" transaction (note the line `blockType: BlockTypeEnum.userSend.index`):

```dart
  factory AccountBlockTemplate.callContract(
          Address toAddress, TokenStandard tokenStandard, int amount, List<int> data) =>
      AccountBlockTemplate(
          blockType: BlockTypeEnum.userSend.index,
          toAddress: toAddress,
          tokenStandard: tokenStandard,
          amount: amount,
          data: data);
```

That template transaction then needs to be fleshed out with all the other relevant information (nonce, signature, etc.), and signed, to be broadcast to the network in json format:

```dart
  Map<String, dynamic> toJson() {
    final j = <String, dynamic>{};

    j['version'] = version;
    j['chainIdentifier'] = chainIdentifier;
    j['blockType'] = blockType;
    j['hash'] = hash.toString();
    j['previousHash'] = previousHash.toString();
    j['height'] = height;
    j['momentumAcknowledged'] = momentumAcknowledged.toJson();
    j['address'] = address.toString();
    j['toAddress'] = toAddress.toString();
    j['amount'] = amount;
    j['tokenStandard'] = tokenStandard.toString();
    j['fromBlockHash'] = fromBlockHash.toString();
    j['data'] = BytesUtils.bytesToBase64(data);
    j['fusedPlasma'] = fusedPlasma;
    j['difficulty'] = difficulty;
    j['nonce'] = nonce;
    j['publicKey'] = BytesUtils.bytesToBase64(publicKey);
    j['signature'] = BytesUtils.bytesToBase64(signature);
    return j;
  }
```

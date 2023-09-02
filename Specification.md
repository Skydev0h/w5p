# Extensible Wallet V5

Author: Oleg Andreev <oleg@tonkeeper.com>

This is an extensible wallet specification aimed at replacing V4 and allowing arbitrary extensions.

* [Features](#features)
* [Overview](#overview)
* [Discussion](#discussion)
* [Wallet ID](#wallet-id)
* [Packed address](#packed-address)
* [TL-B definitions](#tl-b-definitions)
* [Source code](#source-code)


## Credits

Thanks to [Andrew Gutarev](https://github.com/pyAndr3w) for the idea to set c5 register to a [list of pre-composed actions](https://github.com/pyAndr3w/ton-preprocessed-wallet-v2).

Thanks to [@subden](https://t.me/subden), [@botpult](https://t.me/botpult) and [@tvorogme](https://t.me/tvorogme) for ideas and discussion.


## Features

* Arbitrary amount of outgoing messages is supported via action list.
* Wallet code can be upgraded transparently without breaking user's address in the future.
* Unlimited number of plugins can be deployed sharing the same code.
* Wallet code can be extended by anyone in a decentralized and conflict-free way: multiple feature extensions can co-exist.
* Extensions can perform the same operations as the signer: emit arbitrary messages on behalf of the owner, add and remove extensions.
* Signed requests can be delivered via internal message to allow 3rd party pay for gas.
* Extensible ABI for future additions.

## Overview

Wallet V5 supports **2 authentication modes**, all standard output actions (send message, set library, code replacement) plus additional **3 operation types**.

Authentication:
* by signature
* by extension

Operation types:
* standard output actions
* “set data”
* install extension
* remove extension

Signed messages can be delivered both by external and internal messages.

All operation types are available to all authentication modes.

## Discussion

### What is the job of the wallet?

The job of the wallet is to send messages to other apps in the TON network on behalf of a single user identified by a single public key.
User may delegate this job to other apps via extensions.

### The wallet is not for:

* multi-user operation: you should use a multisig or DAO solution instead.
* routing of incoming payments and messages: use a specialized contract instead.
* imposing limits on access to certain assets: put account restriction inside a jetton, or use a lockup contract instead.

### Extending the wallet

**A. Use extensions**

The best way to extend functionality of the wallet is to use the extensions mechanism that permit delegating access to the wallet to other contracts.

From the perspective of the wallet, every extension can perform the same actions as the owner of a private key. Therefore limits and capabilities can be embedded in such an extension with a custom storage scheme.

Extensions can co-exist simultaneously, so experimental capabilities can be deployed and tested independently from each other.

**B. Code optimization**

Backwards compatible code optimization **can be performed** with a single `set_code` action (`action_set_code#ad4de08e`) signed by the user. That is, hypothetical upgrade from `v5R1` to `v5R2` can be done in-place without forcing users to change their wallet address.

If the optimized code requires changes to the data layout (e.g. reordering fields) the user can sign a request with two actions: `set_code` (in the standard action) and `set_data` (an extended action per this specification). Note that `set_data` action must make sure `seqno` is properly incremented after the upgrade as to prevent replays. Also, `set_data` must be performed right before the standard actions to not get overwritten by extension actions.

User agents **should not** make `set_code` and `set_data` actions available via general-purpose API to prevent misuse and mistakes. Instead, they should be used as a part of migration logic for a specific wallet code.

To restore the wallet by a seed phrase, user agent should use the original code and should expect the upgraded code to work in exactly the same way as previously.

**C. Emergency upgrades**

This is a variant of (B), but may require modifying some functionality as to prevent user from suffering loss of funds.

**D. Substantial upgrades**

We **do not recommend** performing substantial wallet upgrades in-place using `set_code`/`set_data` actions. Instead, user agents should have support for multiple accounts and easy switching between them.

In-place migration requires maintaining backwards compatibility for all wallet features, which in turn could lead to increase in code size and higher gas and rent costs.


### Can the wallet outsource payment for gas fees?

Yes! You can deliver signed messages via an internal message from a 3rd party wallet. Also, the message is handled exactly like an external one: after the basic checks the wallet takes care of the fees itself, so that 3rd party does not need to overpay for users who actually do have TONs.

### Does the wallet grow with number of plugins?

Not really. Wallet only accumulates code extensions. So if even you have 100500 plugins based on just three types of contracts, your wallet would only store extra ≈96 bytes of data.

### Can plugins implement subscriptions that collect tokens?

Yes. Plugins can emit arbitrary messages, including token transfers, on behalf of the wallet.

### How can a plugin collect funds?

Plugin needs to send a request with a message to its own address.

### How can a plugin self-destruct?

Plugin does not need to remove its extension code from the wallet — they can simply self-destroy by sending all TONs to the wallet with sendmode 128.

### How can I deploy a plugin, install its code and send it a message in one go?

You need two requests in your message body: first one installs the extension code, the second one sends raw message to your plugin address.

### How does the wallet know which plugins it has installed?

Extension contracts are designed in such way that each one checks that it was deployed by its proper wallet. For an example of this initialization pattern see how NFT items or jetton wallets do that. 

Your wallet can only trust the extension code that was audited to perform such authenticated initialization. Users are not supposed to install arbitrary extensions unknown to the user agent.


## Wallet ID

Wallet ID disambiguates requests signed with the same public key to different wallet versions (V3/V4/V5) or wallets deployed on different chains.

For Wallet V5 we suggest using the following wallet ID:

```
mainnet: 20230823 + workchain
testnet: 30230823 + workchain
```

## Packed address

To make authorize extensions efficiently we compress 260-bit address (workchain + sha256 of stateinit) into a 256-bit integer:

```
int addr = addr_hash ^ (wc + 1)
```

Previously deployed wallet v4 was packing the address into a cell which costs ≈500 gas, while access to dictionary costs approximately `120*lg2(N)` in gas, that is serialization occupies more than half of the access cost for wallets with up to 16 extensions. This design makes packing cost around 50 gas and allows cutting the authentication cost 2-3x for reasonably sized wallets.

As of 2023 TON network consists of two workchains: -1 (master) and 0 (base). This means that the proposed address packing reduces second-preimage resistance of sha256 by 1 bit which we consider negligible. Even if the network is expanded with 254 more workchains in a distant future, our scheme would reduce security of extension authentication by only 8 bits down to 248 bits. Note that birthday attack is irrelevant in our setting as the user agent is not installing random extensions, although the security margin is plenty anyway (124 bits).



## TL-B definitions

Action types:

```tlb
// Standard actions from block.tlb:
out_list_empty$_ = OutList 0;
out_list$_ {n:#} prev:^(OutList n) action:OutAction
  = OutList (n + 1);
action_send_msg#0ec3c86d mode:(## 8) 
  out_msg:^(MessageRelaxed Any) = OutAction;
action_set_code#ad4de08e new_code:^Cell = OutAction;
action_reserve_currency#36e6b809 mode:(## 8)
  currency:CurrencyCollection = OutAction;
libref_hash$0 lib_hash:bits256 = LibRef;
libref_ref$1 library:^Cell = LibRef;
action_change_library#26fa1dd4 mode:(## 7) { mode <= 2 }
  libref:LibRef = OutAction;

// Extended actions in W5:
action_list_basic$0 {n:#} actions:^(OutList n) = ActionList n 0;
action_list_extended$1 {m:#} {n:#} prev:^(ActionList n m) action:ExtendedAction = ActionList n (m+1);

action_set_data#1ff8ea0b data:^Cell = ExtendedAction;
action_add_ext#1c40db9f addr:MsgAddressInt = ExtendedAction;
action_delete_ext#5eaef4a4 addr:MsgAddressInt = ExtendedAction;
```

Authentication modes:

```tlb
signed_request$_ 
  signature:    bits512                   // 512
  subwallet_id: uint32                    // 512+32
  valid_until:  uint32                    // 512+32+32
  msg_seqno:    uint32                    // 512+32+32+32 = 608
  inner: InnerRequest = SignedRequest;

internal_signed#7369676E signed:SignedRequest = InternalMsgBody;
internal_extension#6578746E inner:InnerRequest = InternalMsgBody;
external_signed#7369676E signed:SignedRequest = ExternalMsgBody;

actions$_ {m:#} {n:#} actions:(ActionList n m) = InnerRequest;
```


## Source code

See [contracts/wallet_v5_3.fc].
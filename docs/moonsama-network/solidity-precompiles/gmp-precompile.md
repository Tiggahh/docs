# GMP Precompile


## Introduction

Moonsama Network Routed Liquidity (MRL) refers to Moonbeam’s use case as the port parachain for liquidity from origin chains into other Polkadot parachains. This is possible because of general message passing (GMP), where messages with arbitrary data and tokens can be sent across non-parachain blockchains through [chain-agnostic GMP protocols](builders/interoperability/protocols). These GMP protocols can combine with [Polkadot's XCM messaging system](builders/interoperability/xcm/overview) to allow for seamless liquidity routing.

The GMP precompile acts as an interface for Moonsama Network Routed Liquidity, acting as a middleman between token-bearing messages from GMP protocols and parachains connected to Moonsama Network via [XCMP](builders/interoperability/xcm/overview/#xcm-transport-protocols). Currently the GMP Precompile only supports the relaying of liquidity through the [Wormhole GMP protocol](builders/interoperability/protocols/wormhole).

The GMP Precompile is only available on  and is located at the following address:



```
0x0000000000000000000000000000000000000816

```

In practice, it is unlikely that a developer will have to directly interact with the precompile. GMP protocols' relayers interact with the precompile to complete cross-chain actions, so the origin chain that the cross-chain action originates is where the developer has the responsibility to ensure that the GMP precompile is used *eventually*.

## The GMP Solidity Interface

`[Gmp.sol](/blob/master/precompiles/gmp/Gmp.sol)` is a Solidity interface that allows developers to interact with the precompile:

- **wormholeTransferERC20**(*bytes memory* vaa) - receives a Wormhole bridge transfer [verified action approval (VAA)](https://book.wormhole.com/wormhole/4_vaa.html), mints tokens via the Wormhole token bridge, and forwards the liquidity to the custom payload’s [multilocation](builders/interoperability/xcm/overview/#general-xcm-definitions)
- VAAs are payload-containing packages generated after origin-chain transactions and are discovered by Wormhole [guardian network spies](https://book.wormhole.com/wormhole/6_relayers.html?search=#specialized-relayers). The payload is expected to be a precompile-specific SCALE encoded object, as explained in this guide's [Wormhole section](builders/pallets-precompiles/precompiles/gmp/#building-the-payload-for-wormhole)

The most common instance that a user will have to interact with the precompile is in the case of a recovery, where a relayer doesn’t complete an MRL transaction. For example, a user would have to search for the VAA that comes with their origin chain transaction and then manually invoke the `wormholeTransferERC20` function.

## Building the Payload for Wormhole

Currently the GMP precompile only supports sending liquidity with Wormhole, through Moonsama Network, and into other parachains. The GMP precompile does not assist with a route from parachains back to Moonsama Network and subsequently Wormhole connected chains.

To send liquidity from a Wormhole-connected origin chain like Ethereum, users must invoke the `[transferTokensWithPayload` method](<https://book.wormhole.com/technical/evm/tokenLayer.html#contract-controlled-transfer>) on the [origin-chain's deployment](https://book.wormhole.com/reference/contracts.html#token-bridge) of the [WormholeTokenBridge smart contract](https://github.com/wormhole-foundation/wormhole/blob/main/ethereum/contracts/bridge/interfaces/ITokenBridge.sol). This function requires a bytes payload, which must be formatted as a SCALE encoded multilocation object wrapped within [another precompile-specific versioned type](/blob/1d664f3938698a6cd341fb8f36ccc4bb1104f1ff/precompiles/gmp/src/types.rs#L25-L39).

You may be unfamiliar with both SCALE encoding and multilocations if you are not familiar with the Polkadot ecosystem. [SCALE encoding](https://docs.substrate.io/reference/scale-codec/) is a compact form of encoding that Polkadot uses. The `[MultiLocation` type](<https://wiki.polkadot.network/docs/learn-xcvm>) is used to define a relative point in Polkadot, such as a specific account on a specific parachain (Polkadot blockchain).

Moonbeam’s GMP protocol requires a multilocation to represent the destination for liquidity routing, which most likely means an account on some other parachain. Whatever it is, this destination must be expressed as relative to Moonsama Network.

Remember

Multilocations being relative is important, because a parachain team may erroneously give you a multilocation relative to their own chain, which can be different. Providing an incorrect multilocation can result in **loss of funds**!

Each parachain will have their own methods of interpreting a multilocation, and should confirm with the project that the multilocation that you form is correct. That being said, it is most likely that you will be forming a multilocation with an account.

There are multiple types of accounts that can be included in a multilocation, which you must know beforehand when constructing your multilocation. The two most common are:

- **AccountKey20** — an account ID that is 20-bytes in length, including Ethereum-compatible account IDs such as those on Moonsama Network
- **AccountId32** — an account ID that is 32-bytes in length, standard in Polkadot and its parachains

The following multilocation templates target accounts on other parachains with Moonsama Network as the relative origin. To use them, replace `INSERT_PARACHAIN_ID` with the parachain ID of the network you wish to send funds to and replace `ADDRESS_HERE` with the address of the account you want to send funds to on that parachain.

AccountId32AccountKey20

```
{
    "parents": 1,
    "interior": {
        "X2": [
            { "Parachain": INSERT_PARACHAIN_ID },
            {
                "AccountId32": {
                    "id": "ADDRESS_HERE"
                }
            }
        ]
    }
}

```

It can be difficult to correctly SCALE encode the entire payload without the right tools, especially due to the [custom types expected by the precompile](/blob/1d664f3938698a6cd341fb8f36ccc4bb1104f1ff/precompiles/gmp/src/types.rs#L25-L39). Fortunately, there are Polkadot JavaScript packages that can assist with this, such as `[@polkadot/types](https://www.npmjs.com/package/@polkadot/types)`. The following script shows how to create a `Uint8Array` that can be used as a payload for the GMP precompile:

```
import { TypeRegistry, Enum, Struct } from '@polkadot/types';

// Creates a type registry to properly work with the precompile's input types
const registry = new TypeRegistry();

// Define the precompile's input types VersionedUserAction and XcmRoutingUserAction
// /blob/1d664f3938698a6cd341fb8f36ccc4bb1104f1ff/precompiles/gmp/src/types.rs#L25-L39
class VersionedUserAction extends Enum {
 constructor(value) {
   super(registry, { V1: XcmRoutingUserAction }, value);
 }
}
class XcmRoutingUserAction extends Struct {
 constructor(value) {
   super(registry, { destination: 'MultiLocation' }, value);
 }
}

// A function that creates a SCALE encoded payload to use with transferTokensWithPayload
function createMRLPayload(parachainId, account, isEthereumStyle) {
  // Create a multilocation object based on the target parachain's account type
  const multilocation = {
    parents: 1,
    interior: {
      X2: [
        { Parachain: parachainId },
        isEthereumStyle ?
          { AccountKey20: { key: account } } :
          { AccountId32: { id: account }
      }]
    }
  };

  // Format multilocation object as a Polkadot.js type
  const destination = registry.createType('MultiLocation', multilocation);

  // Wrap and format the MultiLocation object into the precompile's input type
  const userAction = new XcmRoutingUserAction({ destination });
  const versionedUserAction = new VersionedUserAction({ V1: userAction });

  // SCALE encode resultant precompile formatted objects
  return versionedUserAction.toU8a();
}

```

## Restrictions

The GMP precompile is currently in its early stages. There are many restrictions, and it only supports a “happy path” into parachains. Here are some restrictions that you should be aware of:

- There is currently no fee mechanism. Relayers that run the forwarding of liquidity on Moonsama Network to a parachain will be subsidizing transactions. This may change in the future
- The precompile does not check to ensure that the destination chain supports the token that is being sent to it. **Incorrect multilocations may result in loss of funds**
- Errors in constructing a multilocation will result in reverts, which will trap tokens and a loss of funds
- There is currently no recommended path backwards, from parachains to other chains like Ethereum. There is additional protocol level work that must be done before a one-click method can be realized
    - Due to a restriction with the ERC-20 XC-assets, the only way to send tokens from a parachain back through Moonsama Network is to have xcGLMR on the origin parachain and use it as a fee asset when sending tokens back
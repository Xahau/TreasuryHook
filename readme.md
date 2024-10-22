### Treasury funds: hook

Voluntarily lock up the amount of **XAH** going out from the treasury account every set ledger interval.

The idea is to blackhole the treasury account, and the only way to withdraw funds is through the invoke transaction. Anybody can invoke the claim reward and the withdraw transactions on treasury account - after specified ledger interval.

### Installing the Hook on your account

```
import { Client, Wallet, SetHookFlags, AccountSetAsfFlags } from "@transia/xrpl";
import {
  createHookPayload,
  setHooksV3,
  SetHookParams,
  flipHex
} from "@transia/hooks-toolkit";

import {
  uint32ToHex,
  xflToHex,
  xrpAddressToHex,
} from '@transia/hooks-toolkit/dist/npm/src/libs/binary-models'

import { accountSet } from '@transia/hooks-toolkit/dist/npm/src/libs/xrpl-helpers'

export async function main(): Promise<void> {
  const serverUrl = "wss://xahau-test.net";   // Xahau Test Network
  const client = new Client(serverUrl);
  await client.connect();
  client.networkID = await client.getNetworkID();
  const secret = "" // Your account secret
  const hookWallet = Wallet.fromSeed(secret);

  const amount_in_xfl = xflToHex(10);
  const ledger_interval = flipHex(uint32ToHex(100));
  const destination_account = xrpAddressToHex("rHdkzpxr3VfabJh9tUEDv7N4DJEsA4UioT");

  accountSet(client, hookWallet, AccountSetAsfFlags.asfTshCollect);

  const hook1 = createHookPayload({
    version: 0,
    createFile: "treasuryInvoke",
    namespace: "TREASURY",
    flags: SetHookFlags.hsfOverride,
    "HookParameters": [
        HookParameter: {
                    HookParameterName: "41", // 41 is the hex value for 'A'
                    HookParameterValue: amount_in_xfl
        },
        HookParameter: {
                    HookParameterName: "4C", // 4C is the hex value for 'L'
                    HookParameterValue: ledger_interval
        },
        HookParameter: {
                    HookParameterName: "44", // 44 is the hex value for 'D'
                    HookParameterValue: destination_account
        }
    ],
    hookOnArray: ["Invoke"]
  });

  const hook2 = createHookPayload({
    version: 0,
    createFile: "genesisMint",
    namespace: "GENESISMINT",
    flags: SetHookFlags.hsfCollect + SetHookFlags.hsfOverride,
    "HookParameters": [
        HookParameter: {
                    HookParameterName: "44", // 44 is the hex value for 'D'
                    HookParameterValue: destination_account
        }
    ],
    hookOnArray: ["GenesisMint"],
  });


  await setHooksV3({
    client: client,
    seed: hookWallet.seed,
    hooks: [{ Hook: hook1 }, { Hook: hook2 }],
  } as SetHookParams);

  await client.disconnect();
}

main();
```

### Converting ledger interval of 100 to 4 byte little endian uint32

1. Represent 100 in hexadecimal:
   Decimal 100 is 0x64 in hexadecimal.

2. Pad the hexadecimal representation to 4 bytes:
   A 4-byte (32-bit) value requires 8 hex digits. So, we pad 0x64 to 0x00000064.
3. Arrange the bytes in little-endian order (least significant byte first):
   Original 4-byte hex value: 00 00 00 64
   Little-endian order: 64 00 00 00

   Thus, 100 in little-endian 4-byte uint32 is represented as: `64 00 00 00` i.e., `64000000`

### How it works

1. A person with Account A can create another account B, and install the treasury hook on account B (lets call it treasury account), with account A as the hook parameter(0x44U or D for destination account). Other hook parameters: limit amount to withdraw and the ledger interval to withdraw the limit amount.

2. Make some invoke transactions on treasury account, and check if the hook paramters are set correctly.

3. Now you can transfer the funds in Account A to treasury account.

4. Since treasury account has all the funds, it can also claim rewards every month.

   - The monthly claim rewards(4%) will be immediately transferred to account A.(Someone has to invoke the claim reward transaction on treasury account)
   - And account A can also withdraw set amount(set as hook parameter - 0x41U or A) from treasury account every specified ledger interval(set as hook parameter - 0x4CU or L).

5. Once everything is working fine, blackhole the treasury account.

This way the amount of XAH inflow into the market becomes predictable.

### Important notes

1. The first hook (the treasury hook) must hook on Invoke.
2. The second hook (the genesis mint hook) must hook only on GenesisMint.

If the genesis mint hook is not set, the treasury will not be able to forward the claim reward to the destination account.

### Tools

1. amount to xfl: https://hooks.services/tools/float-to-xfl
2. raddress to account id: https://hooks.services/tools/raddress-to-accountid

## Invoke Transactions:

**Withdraw amount from treasury:** Note that the amount must be in xfl, and it should be less than the limit set by the treasury hook.

```
{
  "TransactionType": "Invoke",
  "Account": account.raddress,
  "Fee": "1035",
  "HookParameters": [
    {
      "HookParameter": {
        "HookParameterName": "57", // 57 is the hex value for 'W' (withdraw)
        "HookParameterValue": "00008D49FD1A8754"
      }
    }
  ]
}
```

**Claim monthly rewards:**

```
{
  "TransactionType": "Invoke",
  "Account": account.raddress,
  "Fee": "1028",
  "HookParameters": [
    {
      "HookParameter": {
        "HookParameterName": "43", // 43 is the hex value for 'C' (claim)
        "HookParameterValue": "00"
      }
    }
  ]
}
```

Abstract diagram to illustrate the concept:
<img width="1007" alt="Screenshot 2024-10-22 at 1 27 16 PM" src="https://github.com/user-attachments/assets/414116e6-df26-469f-8e29-136db4b6dfb7">

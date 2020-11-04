# SDK
> Please note that HermezJS is still under development and hasn't been released yet. We provide this tutorial as an overview of what is to come.

HermezJS is an open source SDK to interact with Hermez Rollup network.  

## SDK Howto
In this tutorial we will walk through the process of using the SDK to:
1. Creating a wallet
2. Making a deposit from Ethereum into the Hermez network
3. Making transfers
4. Checking transactions' status
5. Withdrawing funds back to Ethereum network.

### Import modules
Load hermezjs library.

```js
import hermez from './src/index.js'

```

### Create Transaction Pool
Initialize some local storage where user transactions are stored. This step must be done before requesting any transaction to Hermez network.

```js
  hermez.initializeTransactionPool()
```

### Connect to Ethereum Network
Some of the operations in Hermez network, such as sending L1 transactions, require interacting with smart contracts. It is thus necessary to initialize a Ethereum provider. In this example we connect to localhost, but we could instead connect to 'rinkeby' network once the contracts are deployed.

```js
  hermez.Providers.setProvider('http://localhost:8545')
```

### Create a Wallet 
We can create a new Hermez wallet by providing the Ethereum account index associated with the procider initialized. This wallet will store the Ethereum and Babyjubjub keys for the Hermez account. As discussed in the [`developer-guide`](../developers/dev-guide?id=Accounts), the Ethereum address is used to authorize L1 transactions, and the Babyjubjub key is used to authorize L2 transactions.

```js
  const {hermezWallet, hermezEthereumAddress } = await hermez.BabyJubWallet.createWalletFromEtherAccount(0)
```
### Check Token exists in Hermez Network
Before being able to operate on the Hermez network, we must ensure that the token we want to operate with is listed. For that we make a call to the Hermez Coordinator API that will list all listed tokens. All tokens in Hermez Network must be ERC20.

```js
  const tokens = await hermez.CoordinatorAPI.getTokens()
  console.log(tokens)

>>>>
  {
    "tokens": [
      {
        "id": 4444,
        "ethereumAddress": "0xaa942cfcd25ad4d90a62358b0dd84f33b398262a",
        "name": "Maker Dai",
        "symbol": "DAI",
        "decimals": 18,
        "ethereumBlockNum": 539847538,
        "USD": 1.01,
        "fiatUpdate": null
      }
    ],
    "pagination": { "totalItems": 2048, "firstItem": 50, "lastItem": 2130 }
  }
```

In this case, Hermez network only DAI as the only available token to operate.
**NOTE** By default, the SDK connects with the boot coordinator.


### Deposit Tokens from Ethereum into Hermez Network
Creating an Hermez account and depositing tokens are done simultaneously as a L1 transaction.  In this example we are going to deposit 100.0 DAI to the Hermez account. The steps are:
1. Select amount to deposit from Ethereum into hermez using getTokenAmountBigInt
2. Select the token denomination of the deposit. Hermez contains a list of supported token that can be queried with getTokens(). This function returns the list of supported tokens. Only tokens in this list can be used.

```js
  // Configure deposit amount
  const amount = hermez.Utils.getTokenAmountBigInt('10020',2)

  // retrieve DAI token info from Hermez network
  tokenDAI = token.tokens[0]

  // make deposit of ERC20 Tokens
  await hermez.deposit(amount,
                       hermezEthereumAddress,
                       tokenDAI, 
                       hermezWallet.publicKeyCompressedHex)
```
Internally, the deposit funcion calls Hermez smart contract to add the L1 transaction.

### Verify Balance
A token balance can be obtained by querying a Hermez node and passing the hermezEthereumAddress of the Hermez account.

```js
  const acountInfo = await hermez.CoordinatorAPI.getAccounts(hermezEthereumAddress)
  console.log(accountInfo)

>>>>>

{
  accounts: [
    {
      accountIndex: 'hez:DAI:4444',
      nonce: 121,
      balance: '8708856933496328593',
      bjj: 'hez:rR7LXKal-av7I56Y0dEBCVmwc9zpoLY5ERhy5w7G-xwe',
      hezEthereumAddress: 'hez:0xaa942cfcd25ad4d90a62358b0dd84f33b398262a',
      token: [Object]
    }
  ],
  pagination: { totalItems: 2048, firstItem: 50, lastItem: 2130 }
}

```
### Transfer
At this point, a Hermez account is already created with 100.2 DAI tokens. The next step is to transfer some funds to another Hermez rollup account.

First we create a second wallet following the same procedure we saw earlier:

```js
   const {hermezWallet2, hermezEthereumAddress2 } =
                      await hermez.BabyJubWallet.createWalletFromEtherAccount(1)
```

Next step we compute the fees for the transaction. For this we consult the recommended fees from the coordinator state.

```js
  // fee computation
  const state = await hermez.CoordinatorAPI.getState()
  console.log(fees)

>>>>
{
  existingAccount: 0.1,
  createAccount: 1.3,
  createAccountInternal: 0.5
}

```

The returned fees are the suggested feeds for different transactions:
- existingAccount : Make a transfer to an existing account
- createAccount   : Make a transfer to a inexistent account, and create a Reguler account
- createAccountInternal : Make a transfer to a non-existent account and create internal account

The fee amounts are given in USD. However, fees are payed in the token of the transaction. So, we need to do a conversion.

```js
  const usdTokenExchangeRate = tokenDAI.USD
  const fee = fees.existingAccount / usdTokenExchangeRate
```

The last part is to request the actual transfer.

```js
  // src account
  const from = (await hermez.CoordinatorAPI.getAccounts(hermezEthereumAddress)).accounts[0]
  // dst account
  const to = (await hermez.CoordinatorAPI.getAccounts(hermezEthereumAddress2)).accounts[0]
  // amount to transfer
  const newAmount = hermez.Utils.getTokenAmountBigInt('10',2)

  // generate L2 transaction
  const {transaction, encodedTransaction} = await hermez.TxUtils.generateL2Transaction(
    {
      from: from.accountIndex,
      to: to.accountIndex,
      amount: hermez.Float16.float2Fix(hermez.Float16.floorFix2Float(newAmount)),
      fee,
      nonce: from.nonce
    },
    hermezWallet.publicKeyCompressedHex, from.token)

  // sign encoded transaction
  hermezWallet.signTransaction(transaction, encodedTransaction)
  // send transaction to coordinator
  const result = await hermez.Tx.send(transaction, hermezWallet.publicKeyCompressedHex)
  
  console.log(result)

>>>>>

{ status: 200, id: '0x00000000000001e240004700', nonce: 122 }


```
The result status 200 shows that transaction has been correctly received. Additionally, we receive the a nonce matching the transaction we sent, and an id that we can use to verify the status of the transaction,

### Verifying Transaction status
Transactions received by the operator will be stored in its transaction pool while they haven't been processed. To check a transaction in the transaction pool we make a query to the coordinator node.

```js
  const txInfo = await hermez.CoordinatorAPI.getPoolTransaction(result.id)
  console.log(txInfo)

>>>>

{
  id: '0x00000000000001e240004700',
  type: 'Transfer',
  fromAccountIndex: 'hez:DAI:4444',
  toAccountIndex: 'hez:DAI:309',
  toHezEthereumAddress: 'hez:0xbb942cfcd25ad4d90a62358b0dd84f33b3982699'
  toBjj: 'hez:HVrB8xQHAYt9QTpPUsj3RGOzDmrCI4IgrYslTeTqo6Ix',
  amount: '6303020000000000000',
  fee: 36,
  nonce: 122,
  state: 'pend',
  signature: '72024a43f546b0e1d9d5d7c4c30c259102a9726363adcc4ec7b6aea686bcb5116f485c5542d27c4092ae0ceaf38e3bb44417639bd2070a58ba1aa1aab9d92c03',
  timestamp: '2019-08-24T14:15:22Z',
  batchNum: 5432,
  requestFromAccountIndex: 'hez:0xaa942cfcd25ad4d90a62358b0dd84f33b398262a',
  requestToAccountIndex: 'hez:DAI:33',
  requestToHezEthereumAddress: 'hez:0xbb942cfcd25ad4d90a62358b0dd84f33b3982699',
  requestToBJJ: 'hez:HVrB8xQHAYt9QTpPUsj3RGOzDmrCI4IgrYslTeTqo6Ix',
  requestTokenId: 4444,
  requestAmount: 'string',
  requestFee: 8,
  requestNonce: 6,
  token: {
    id: 4444,
    ethereumAddress: '0xaa942cfcd25ad4d90a62358b0dd84f33b398262a',
    name: 'Maker Dai',
    symbol: 'DAI',
    decimals: 18,
    ethereumBlockNum: 539847538,
    USD: 1.01,
    fiatUpdate: null
  }
}
 
```

At this point, the transactions is still in the coordinator's transaction pool as we can see in the `state` field. There are 4 possible states:
1. **pend** : Pending
2. **fging** : Forging
3. **fged** : Forged
4. **invl** : Invalid

. After a few seconds, we verify transaction status

```js
    // Get transaction confirmation
    const txConf = await hermez.CoordinatorAPI.getHistoryTransaction(txInfo.id)
    console.log(txConf)

>>>>>

{
  L1orL2: 'L2',
  id: '0x00000000000001e240004700',
  itemId: 0,
  type: 'Transsfer',
  position: 5,
  fromAccountIndex: 'hez:DAI:4444',
  toAccountIndex: 'hez:DAI:672',
  amount: '4903020000000000000',
  batchNum: 5432,
  historicUSD: 49.7,
  timestamp: '2019-08-24T14:15:22Z',
  token: {
    id: 4444,
    ethereumAddress: '0xaa942cfcd25ad4d90a62358b0dd84f33b398262a',
    name: 'Maker Dai',
    symbol: 'DAI',
    decimals: 18,
    ethereumBlockNum: 539847538,
    USD: 1.01,
    fiatUpdate: null
  },
  L1Info: null,
  L2Info: {
    "fee": 36
    "historicFeeUSD": 0.1
    "nonce" : 121
  }
}
```


### Withdrawing Funds from Hermez
This is a 2-step process. First we need to do what we call an **Exit**. This moves the user's funds from their token account to a specific Exit merkle tree. Once the funds are in the Exit tree, we can then withdraw them to an Ethereum L1 account.

### Exit

This is normally a L2 transaction. Thus, it uses the same API as a `transfer`.

```js
// generate L2 transaction
  const {transaction, encodedTransaction} = await hermez.TxUtils.generateL2Transaction(
    {
      from: from.accountIndex,
      to: null,
      amount: hermez.Float16.float2Fix(hermez.Float16.floorFix2Float(newAmount)),
      fee,
      nonce: from.nonce
    },
    hermezWallet.publicKeyCompressedHex, from.token)

  // sign encoded transaction
  hermezWallet.signTransaction(transaction, encodedTransaction)
  // send transaction to coordinator
  const result = await hermez.Tx.send(transaction, hermezWallet.publicKeyCompressedHex)
```

The only difference is that we set the `to` property to `null`.

### Force Exit

This is the L1 equivalent of an Exit. With this option, the Smart Contract forces Coordinators to pick up these transactions before they pick up L2 transactions. Meaning that these transactions will always be picked up.

This is a security measure. We don't expect users to need to make a Force Exit.

```js
const result = await hermez.Tx.forceExit(newAmount from.accountIndex, from.token)
```

### Withdraw

Once the Exit information is in the Exit tree, we can move on to make a Withdraw. First we need to track the transaction as we did with the Transfer above.

```js
const txInfo = await hermez.CoordinatorAPI.getPoolTransaction(result.id)
const txConf = await hermez.CoordinatorAPI.getHistoryTransaction(txInfo.id)
```

Once the transaction has been forged and comes up in the History, we can get its information from the Exit tree.

```js
const exitInfo = await hermez.CoordinatorAPI.getExit(txConf.batchNum, txConf.fromAccountIndex)
```

And with the Exit information, we can now make a withdraw.

```js
const result = await hermez.Tx.withdraw(newAmount, from.accountIndex, from.token, hermezWallet.publicKeyCompressedHex, exitInfo.merkleProof.Root, exitInfo.merkleProof.Siblings)
```

The funds should now appear in the Ethereum account that made the withdraw.
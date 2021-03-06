[![License: GPL v3](https://img.shields.io/badge/License-GPL%20v3-blue.svg)](https://github.com/Conflux-Chain/go-conflux-sdk/blob/master/LICENSE)
[![Documentation](https://img.shields.io/badge/Documentation-GoDoc-green.svg)](https://godoc.org/github.com/Conflux-Chain/go-conflux-sdk)

# Conflux Golang API

The Conflux Golang API allows any Golang client to interact with a local or remote Conflux node based on JSON-RPC 2.0 protocol. With Conflux Golang API, user can easily manage accounts, send transactions, deploy smart contracts and query blockchain information.

## Install
```
go get github.com/Conflux-Chain/go-conflux-sdk
```
You can also add the Conflux Golang API into vendor folder.
```
govendor fetch github.com/Conflux-Chain/go-conflux-sdk
```

## Usage

[api document](https://github.com/Conflux-Chain/go-conflux-sdk/blob/master/api.md)

## Manage Accounts
Use `AccountManager` struct to manage accounts at local machine.
- Create/Import/Update/Delete an account.
- List all accounts.
- Unlock/Lock an account.
- Sign a transaction.

## Query Conflux Information
Use `Client` struct to query Conflux blockchain information, such as block, epoch, transaction, receipt. Following is an example to query the current epoch number:
```go
package main

import (
	"fmt"

	conflux "github.com/Conflux-Chain/go-conflux-sdk"
)

func main() {
	client, err := conflux.NewClient("http://52.175.52.111:12537")
	if err != nil {
		fmt.Println("failed to create client:", err)
		return
	}
	defer client.Close()

	epoch, err := client.GetEpochNumber()
	if err != nil {
		fmt.Println("failed to get epoch number:", err)
		return
	}

	fmt.Println("Current epoch number:", epoch)
}
```

## Send Transaction
To send a transaction, you need to sign the transaction at local machine, and send the signed transaction to local or remote Conflux node.
- Sign a transaction with unlocked account:

    `AccountManager.SignTransaction(tx UnsignedTransaction)`

- Sign a transaction with passphrase for locked account:

	`AccountManager.SignTransactionWithPassphrase(tx UnsignedTransaction, passphrase string)`

- Send a unsigned transaction

    `Client.SendTransaction(tx *types.UnsignedTransaction)`

- Send a encoded transaction

    `Client.SendRawTransaction(rawData []byte)`

- Encode a encoded unsigned transaction with signature and send transaction

    `Client.SignEncodedTransactionAndSend(encodedTx []byte, v byte, r, s []byte)`

To send multiple transactions at a time, you can unlock the account at first, then send multiple transactions without passphrase. To send a single transaction, you can just only send the transaction with passphrase.

## Deploy/Call Smart Contract
You can use `Client.DeployContract` to deploy a contract or use `Client.GetContract` to get a contract by deployed address. Then you can use the contract instance to operate contract, there are GetData/Call/SendTransaction. Please see [api document](https://github.com/Conflux-Chain/go-conflux-sdk/blob/master/api.md) for detail.

### Contract Example
```go
package main

import (
	"encoding/hex"
	"fmt"
	"io/ioutil"
	"math/big"
	"time"

	sdk "github.com/Conflux-Chain/go-conflux-sdk"
	"github.com/Conflux-Chain/go-conflux-sdk/types"
)

func main() {

	//unlock account
	am := sdk.NewAccountManager("./keystore")
	err := am.TimedUnlockDefault("hello", 300*time.Second)
	if err != nil {
		panic(err)
	}

	//init client
	client, err := sdk.NewClient("http://testnet-jsonrpc.conflux-chain.org:12537")
	if err != nil {
		panic(err)
	}
	client.SetAccountManager(am)

	//deploy contract
	abiPath := "./contract/erc20.abi"
	bytecodePath := "./contract/erc20.bytecode"
	var contract *sdk.Contract

	abi, err := ioutil.ReadFile(abiPath)
	if err != nil {
		panic(err)
	}

	bytecodeHexStr, err := ioutil.ReadFile(bytecodePath)
	if err != nil {
		panic(err)
	}

	bytecode, err := hex.DecodeString(string(bytecodeHexStr))
	if err != nil {
		panic(err)
	}

	doneChan := client.DeployContract(string(abi), bytecode, nil, time.Duration(time.Second*30), func(c sdk.Contractor, txhash *types.Hash, err error) {
		if err != nil {
			panic(err)
		}
		contract = c.(*sdk.Contract)
		fmt.Printf("deploy contract by client.DeployContract done\ncontract address: %+v\ntxhash:%v\n\n", *contract.Address, txhash)
	})

	_ = <-doneChan
	time.Sleep(30 * time.Second)

	//get data for send/call contract method
	data, err := contract.GetData("balanceOf", contract.Address.ToCommonAddress())
	if err != nil {
		panic(err)
	}
	fmt.Printf("get data of method balanceOf is: 0x%x\n\n", *data)

	//call contract method
	user := types.Address("0x19f4bcf113e0b896d9b34294fd3da86b4adf0302")
	balance := &struct{ Balance *big.Int }{}
	err = contract.Call(nil, balance, "balanceOf", user.ToCommonAddress())
	if err != nil {
		panic(err)
	}
	fmt.Printf("address %v balance in contract is: %+v\n\n", user, balance)

	//send transction for contract method
	to := types.Address("0x160ebef20c1f739957bf9eecd040bce699cc42c6")
	txhash, err := contract.SendTransaction(nil, "transfer", to.ToCommonAddress(), big.NewInt(10))
	if err != nil {
		panic(err)
	}

	fmt.Printf("transfer %v erc20 token to %v done, tx hash: %v\n\n", 10, to, txhash)

	fmt.Println("wait for transaction be confirmed...")
	for {
		time.Sleep(time.Duration(1) * time.Second)
		tx, err := client.GetTransactionByHash(*txhash)
		if err != nil {
			panic(err)
		}
		if tx.Status != nil {
			fmt.Printf("transaction is confirmed.")
			break
		}
	}
}
```
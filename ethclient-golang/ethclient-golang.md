# Creating Raw Ethereum Interactions in Go: Blob Transactions, Tracing Transactions, and Others.

The ethclient package from Go-Ethereum (Geth) provides an API wrapper for JSON-RPC requests to the Ethereum network, similar to web3.js and ethers.js.

However, some capabilities of the JSON-RPC, like transaction tracing, are not exposed in API of ethclient (and web3.js and ethers.js).

This tutorial shows how to use ethclient for actions where the ethclient supports the JSON-RPC call and when it does not.

As the diagram below illustrates, sometimes we can accomplish an action by using an API in ethclient, but sometimes we have to craft the RPC call ourselves:

![Diagram of an app using ethclient for a json-rpc call](https://static.wixstatic.com/media/706568_24a0e0b23a0346a28190e237ebeb7ee4~mv2.png/v1/fill/w_592,h_173,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_24a0e0b23a0346a28190e237ebeb7ee4~mv2.png)

At the end of the tutorial, we will show how to execute a blob transaction, which Ethereum recently added support for in the Decun upgrade.

We will also perform some Ethereum transaction related concepts like signing and verifying digital signatures.

## Prerequisites

-   The Go language should be installed on your computer; if not, [**see the download instructions**](https://go.dev/doc/install).
-   Some basic Go programming.

## Getting started

We will use the Sepolia network throughout this tutorial, but what we show will also work on mainnet or other testnets. Be sure to have Sepolia ETH.

### Operations we will do in this tutorial:

-   Fetching the suggested gas price on the network
-   Estimating gas for a transaction
-   Constructing and sending an EIP1559 raw transaction
-   Signing and verifying Ethereum messages
-   Retrieving an account's nonce (number of transaction)
-   Tracing a transaction
-   Finally, sending an EIP-4844 blob transaction

To start, create a new project folder, open it and initialize with:

```go!
go mod init eth-rpc
```

We just created our project module. If that was successful, you should see a 'go.mod' file that looks like this

![Image of code in go.mod file upon successful installation](https://static.wixstatic.com/media/706568_84c858b18c5447818ea40cbd7c71cb0e~mv2.png/v1/fill/w_592,h_82,al_c,lg_1,q_85,enc_auto/706568_84c858b18c5447818ea40cbd7c71cb0e~mv2.png)

Install the necessary dependencies:

```go!
go get -u github.com/ethereum/go-ethereum@v1.13.14
go get github.com/ethereum/go-ethereum/rpc@v1.13.14
```

This will generate a 'go.sum' file.

**Troubleshooting Tip**:

If you encounter module-related issues, try the following:

1.  Delete your 'go.mod' and 'go.sum' files and re-initialize it with `go mod init eth-rpc`.

2.  Run `go mod tidy` to synchronize dependencies.

3.  If the issue remains, clear your module cache with `go clean -modcache` and repeat steps 1 and 2.

Now paste the following code to a 'main.go' file inside the project:

```go!
package main
import "fmt"

const (
    sepoliaRpcUrl = "https://rpc.sepolia.ethpandaops.io" // sepolia rpc url
    mainnetRpcUrl = "https://rpc.builder0x69.io/"        // mainnet rpc url
    from = "0x571B102323C3b8B8Afb30619Ac1d36d85359fb84"
    to = "0x4924Fb92285Cb10BC440E6fb4A53c2B94f2930c5"
    data = "Hello Ethereum!"
    privKey = "2843e08c0fa87258545656e44955aa2c6ca2ebb92fa65507e4e5728570d36662"
    gasLimit = uint64(21500) // adjust this if necessary
    wei = uint64(0) // 0 Wei
)

func main() {

    fmt.Println("using ethclient...")

}
```

We will update the main.go file as we move on.

You can run this with `go run main.go`

Now, let's start creating our project functions.

### 1. Fetching the suggested gas price on the network

With Geth's `ethclient` package we can use the SuggestGasPrice API to set the gas price for our transaction appropriate for current network conditions.

Behind the scenes, this method calls the `eth_gasPrice` JSON-RPC API.

Create a `getGasPrice.go` file in the project directory and paste the following code:

```go!
package main

import (
    "context"
    "fmt"
    "log"
    "github.com/ethereum/go-ethereum/ethclient"
)

// getSuggestedGasPrice connects to an Ethereum node via RPC and retrieves the current suggested gas price.

func getSuggestedGasPrice(rpcUrl string) {

    // Connect to the Ethereum network using the provided RPC URL.
    client, err := ethclient.Dial(rpcUrl)
    if err != nil {
        log.Fatalf("Failed to connect to the Ethereum client: %v", err)
    }

    // Retrieve the currently suggested gas price for a new transaction.
    gasPrice, err := client.SuggestGasPrice(context.Background())
    if err != nil {
        log.Fatalf("Failed to suggest gas price: %v", err)
    }

    // Print the suggested gas price to the terminal.
    fmt.Println("Suggested Gas Price:", gasPrice.String())
}
```

Now update the main function in main.go:

```go!
func main() {
    fmt.Println("using ethclient...")

    getSuggestedGasPrice(sepoliaRpcUrl)
    // get gas price on sepolia testnet. This was just added.

}
```

and run:

```go
go run .
```

The reason we use the command `go run` **.** instead of `go run main.go` is to compile and execute all Go source files within the current directory that belong to the same package. This includes the main.go file (which contains the main function) and any other files, such as the one containing our getSuggestedGasPrice function.

We will be using this command going forward.

After running the command, the suggested gas price should be printed on the terminal. Note that it is in Wei.

![Image of suggested gas price using ethclient](https://static.wixstatic.com/media/706568_9862840b5938498e8c9a92bd5f5c4de5~mv2.png/v1/fill/w_592,h_62,al_c,lg_1,q_85,enc_auto/706568_9862840b5938498e8c9a92bd5f5c4de5~mv2.png)

### 2. Estimate gas usage of a transaction

ethclient also includes an EstimateGas method. It returns an estimate of the amount of gas required to successfully process a transaction.

The EstimateGas method calls the `eth_estimateGas` JSON-RPC API with the constructed message as a parameter.

Create an `estimateGas.go` file and paste the following code:

```go!
package main import (
    "context"
    "log"
    "math/big"
    "strings"
    "github.com/ethereum/go-ethereum"
    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/common/hexutil"
    "github.com/ethereum/go-ethereum/ethclient"
)

// estimateGas tries to estimate the suggested amount of gas that is required to execute a given transaction.

func estimateGas(rpcUrl, from, to, data string, value uint64) uint64 {

    // Establish an RPC connection to the specified RPC url
    client, err := ethclient.Dial(rpcUrl)
     if err != nil {
         log.Fatalln(err)
    }

    var ctx = context.Background()

    var (
        fromAddr = common.HexToAddress(from) // Convert the from address from hex to an Ethereum address.
        toAddr = common.HexToAddress(to) // Convert the to address from hex to an Ethereum address.
        amount = new(big.Int).SetUint64(value) // Convert the value from uint64 to *big.Int.
        bytesData []byte
    )

    // Encode the data if it's not already hex-encoded.
    if data != "" {
        if ok := strings.HasPrefix(data, "0x"); !ok {
        data = hexutil.Encode([]byte(data))
        }

        bytesData, err = hexutil.Decode(data)
        if err != nil {
            log.Fatalln(err)
        }
    }

    // Create a message which contains information about the transaction.
    msg := ethereum.CallMsg{
        From: fromAddr,
        To: &toAddr,
        Gas: 0x00,
        Value: amount,
        Data: bytesData,
    }

    // Estimate the gas required for the transaction.
    gas, err := client.EstimateGas(ctx, msg)
    if err != nil {
        log.Fatalln(err)
    }

    return gas
}
```

```go!
func main() {
    fmt.Println("using ethclient...")

    getSuggestedGasPrice(sepoliaRpcUrl)

    eGas := estimateGas(sepoliaRpcUrl, from, to, data, wei) // This was just added.
    fmt.Println("\nestimate gas for the transaction is:", eGas) // This was just added.
}
```

Run the code with `go run .`

We should get this:

![Image of suggested gas price for a specific transaction using ethclient](https://static.wixstatic.com/media/706568_5725ec7860e342acac1ee48728be4cf3~mv2.png/v1/fill/w_592,h_58,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_5725ec7860e342acac1ee48728be4cf3~mv2.png)

### 3. Constructing an EIP1559 raw transaction

Ethereum raw transactions are transactions in their unprocessed form, encoded using the Recursive Length Prefix (RLP) serialization method.

This encoding technique is used by the Ethereum Execution Layer (EL) to serialize and deserialize data.

The raw transaction data is the encoding of the nonce, recipient address(to), transaction value, data payload, and gas limit.

#### **Transaction types**

When manually creating raw transactions for Ethereum, there are several transaction types to choose from, ranging from the old legacy transaction (also referred to as type 0), with explicit gas price specification to EIP-1559 transactions (type 2), which introduces a base fee, a priority fee (miners tip), and a max fee per gas for better gas price predictability.

The base fee is determined by the network and remains fixed for all transactions within a block. However, it adjusts between blocks based on network congestion. You can influence your transaction's priority by increasing the priority fee (tip) offered to miners.

Additionally, there is the [EIP-2930 transaction (type 1)](https://www.rareskills.io/post/eip-2930-optional-access-list-ethereum) and EIP-4844 blob transactions (type 3, which we will discuss later in this article).

#### **Choosing a transaction type in Go with Geth**

The Geth client through its `types` package, supports these various transaction types. For our purposes, we'll focus on `types.DynamicFeeTx`, which corresponds to the EIP-1559 transaction model.

The whole process does not involve making any JSON-RPC call, we just construct the transaction, sign it and serialize it with the RLP encoding scheme.

Create a `createEIP1559RawTX.go` file and paste the following code:

```go!
package main

import (
    "bytes"
    "context"
    "crypto/ecdsa"
    "encoding/hex"
    "fmt"
    "log"
    "math/big"
    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/common/hexutil"
    "github.com/ethereum/go-ethereum/core/types"
     "github.com/ethereum/go-ethereum/crypto"
    "github.com/ethereum/go-ethereum/ethclient"
    "github.com/ethereum/go-ethereum/params"
)

// createRawTransaction creates a raw EIP-1559 transaction and returns it as a hex string.
func createRawTransaction(rpcURL, to, data, privKey string, gasLimit, wei uint64) string {

     // Connect to the Ethereum client using the provided RPC URL.
    client, err := ethclient.Dial(rpcURL)
    if err != nil {
        log.Fatalln(err)
    }

    // Retrieve the chain ID for the target Ethereum network.
    chainID, err := client.ChainID(context.Background())
    if err != nil {
        log.Fatalln(err)
    }

    // Suggest the base fee for inclusion in a block.
    baseFee, err := client.SuggestGasPrice(context.Background())
    if err != nil {
        log.Fatalln(err)
    }

    // Suggest a gas tip cap (priority fee) for miner incentive.
    priorityFee, err := client.SuggestGasTipCap(context.Background())
    if err != nil {
        log.Fatalln(err)
    }

    // Calculate the maximum gas fee cap, adding a 2 GWei margin to the base fee plus priority fee.
    increment := new(big.Int).Mul(big.NewInt(2), big.NewInt(params.GWei))
    gasFeeCap := new(big.Int).Add(baseFee, increment)
    gasFeeCap.Add(gasFeeCap, priorityFee)

    // Decode the provided private key.
    pKeyBytes, err := hexutil.Decode("0x" + privKey)
    if err != nil {
        log.Fatalln(err)
    }

    // Convert the private key bytes to an ECDSA private key.
    ecdsaPrivateKey, err := crypto.ToECDSA(pKeyBytes)
    if err != nil {
        log.Fatalln(err)
    }

    // Extract the public key from the ECDSA private key.
    publicKey := ecdsaPrivateKey.Public()
    publicKeyECDSA, ok := publicKey.(*ecdsa.PublicKey)

    if !ok {
        log.Fatal("Error casting public key to ECDSA")
    }

    // Compute the Ethereum address of the signer from the public key.
    fromAddress := crypto.PubkeyToAddress(*publicKeyECDSA)
    // Retrieve the nonce for the signer's account, representing the transaction count.

    nonce, err := client.PendingNonceAt(context.Background(), fromAddress)
    if err != nil {
        log.Fatal(err)
    }

    // Prepare data payload.
    var hexData string
    if strings.HasPrefix(data, "0x") {
        hexData = data
    } else {
        hexData = hexutil.Encode([]byte(data))
     }
    bytesData, err := hexutil.Decode(hexData)
     if err != nil {
        log.Fatalln(err)
    }

    // Set up the transaction fields, including the recipient address, value, and gas parameters.
    toAddr := common.HexToAddress(to)
    amount := new(big.Int).SetUint64(wei)
    txData := types.DynamicFeeTx{
        ChainID: chainID,
        Nonce: nonce,
        GasTipCap: priorityFee,
        GasFeeCap: gasFeeCap,
        Gas: gasLimit,
        To: &toAddr,
        Value: amount,
        Data: bytesData,
    }

    // Create a new transaction object from the prepared data.
    tx := types.NewTx(&txData)
    // Sign the transaction with the private key of the sender.
    signedTx, err := types.SignTx(tx, types.LatestSignerForChainID(chainID), ecdsaPrivateKey)

    if err != nil {
        log.Fatalln(err)
    }

    // Encode the signed transaction into RLP (Recursive Length Prefix) format for transmission.
    var buf bytes.Buffer
    err = signedTx.EncodeRLP(&buf)

    if err != nil {
        log.Fatalln(err)
    }

    // Return the RLP-encoded transaction as a hexadecimal string.
    rawTxRLPHex := hex.EncodeToString(buf.Bytes())

    return rawTxRLPHex
}
```

Update the main function in main.go:

```go!
func main() {
    fmt.Println("using ethclient...")

    getSuggestedGasPrice(sepoliaRpcUrl)

    eGas := estimateGas(sepoliaRpcUrl, from, to, data, wei)        fmt.Println("\nestimate gas for the transaction is:", eGas)

    rawTxRLPHex := createRawTransaction(sepoliaRpcUrl, to, data, privKey, gasLimit, wei) // This was just added.
    fmt.Println("\nRaw TX:\n", rawTxRLPHex) // This was just added.
}
```

The raw transaction will be created using the private key stored in the 'privKey' variable. To ensure a successful transaction on the Sepolia testnet, replace it (the private key) with a private key that holds test Sepolia ETH.

Run the code with **`go run .`**

We should get the raw transaction, as shown below:

![image of a raw transcation using ethclient](https://static.wixstatic.com/media/706568_f387400bfe4142a8a76ae10452c8c4f1~mv2.png/v1/fill/w_592,h_127,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_f387400bfe4142a8a76ae10452c8c4f1~mv2.png)


We will propagate a raw transaction to the network in the next section.

### 4. Sending a raw transaction

After creating a raw transaction of any type, we can propagate it to the network with the 'ethclient.SendTransaction' function, which takes in the RLP-decoded raw transaction and makes an `eth_sendRawTransaction` JSON-RPC call.

There is some added code (the Transaction struct convertHexField function) here that is not mandatory but helps with better printing of the transaction result.

Create a `sendRawTX.go` file in the project and paste the code below:

```go!
package main

import (
    "context"
    "encoding/hex"
    "encoding/json"
    "fmt"
    "log"
    "reflect"
    "strconv"
    "time"
    "github.com/ethereum/go-ethereum/core/types"
    "github.com/ethereum/go-ethereum/ethclient"
    "github.com/ethereum/go-ethereum/rlp"
)

// Transaction represents the structure of the transaction JSON.
type Transaction struct {
    Type                 string   `json:"type"`
    ChainID              string   `json:"chainId"`
    Nonce                string   `json:"nonce"`
    To                   string   `json:"to"`
    Gas                  string   `json:"gas"`
    GasPrice             string   `json:"gasPrice,omitempty"`
    MaxPriorityFeePerGas string   `json:"maxPriorityFeePerGas"`
    MaxFeePerGas         string   `json:"maxFeePerGas"`    Value                string   `json:"value"`
    Input                string   `json:"input"`
    AccessList           []string `json:"accessList"`
    V                    string   `json:"v"`
    R                    string   `json:"r"`
    S                    string   `json:"s"`
    YParity              string   `json:"yParity"`
    Hash                 string   `json:"hash"`
    TransactionTime      string   `json:"transactionTime,omitempty"`
    TransactionCost      string   `json:"transactionCost,omitempty"`
}

// sendRawTransaction sends a raw Ethereum transaction.
func sendRawTransaction(rawTx, rpcURL string) {
    rawTxBytes, err := hex.DecodeString(rawTx)
    if err != nil {
        log.Fatalln(err)
    }

    // Initialize an empty Transaction struct to hold the decoded data.
    tx := new(types.Transaction)

    // Decode the raw transaction bytes from hexadecimal to a Transaction struct.
    // This step converts the RLP (Recursive Length Prefix) encoded bytes back into
    // a structured Transaction format understood by the Ethereum client.
    err = rlp.DecodeBytes(rawTxBytes, &tx)
    if err != nil {
        log.Fatalln(err)
    }

    // Establish an RPC connection to the specified RPC url        client, err := ethclient.Dial(rpcURL)
    if err != nil {
        log.Fatalln(err)
    }

    // Propagate the transaction
    err = client.SendTransaction(context.Background(), tx)
    if err != nil {
        log.Fatalln(err)
    }

    // Unmarshal the transaction JSON into a struct
    var txDetails Transaction
    txBytes, err := tx.MarshalJSON()
    if err != nil {
        log.Fatalln(err)
    }
    if err := json.Unmarshal(txBytes, &txDetails); err != nil {
        log.Fatalln(err)
    }

    // Add additional transaction details
    txDetails.TransactionTime = tx.Time().Format(time.RFC822)
    txDetails.TransactionCost = tx.Cost().String()

    // Format some hexadecimal string fields to decimal string
    convertFields := []string{"Nonce", "MaxPriorityFeePerGas", "MaxFeePerGas", "Value", "Type", "Gas"}
    for _, field := range convertFields {
        if err := convertHexField(&txDetails, field); err != nil {
            log.Fatalln(err)
        }
    }

    // Marshal the struct back to JSON
    txJSON, err := json.MarshalIndent(txDetails, "", "\t")
    if err != nil {
        log.Fatalln(err)
    }

    // Print the entire JSON with the added fields
    fmt.Println("\nRaw TX Receipt:\n", string(txJSON))
}

func convertHexField(tx *Transaction, field string) error {

    // Get the type of the Transaction struct
    typeOfTx := reflect.TypeOf(*tx)

    // Get the value of the Transaction struct
    txValue := reflect.ValueOf(tx).Elem()

    // Parse the hexadecimal string as an integer
    hexStr := txValue.FieldByName(field).String()

    intValue, err := strconv.ParseUint(hexStr[2:], 16, 64)
    if err != nil {
        return err
    }

    // Convert the integer to a decimal string
    decimalStr := strconv.FormatUint(intValue, 10)

    // Check if the field exists
    _, ok := typeOfTx.FieldByName(field)
    if !ok {
        return fmt.Errorf("field %s does not exist in Transaction struct", field)
    }

    // Set the field value to the decimal string
    txValue.FieldByName(field).SetString(decimalStr)

    return nil
}
```

Now update the main function in main.go:

```go!
func main() {
    fmt.Println("using ethclient...")

    getSuggestedGasPrice(sepoliaRpcUrl)

    eGas := estimateGas(sepoliaRpcUrl, from, to, data, wei)
    fmt.Println("\nestimate gas for the transaction is:", eGas)

    rawTxRLPHex := createRawTransaction(sepoliaRpcUrl, to, data, privKey, gasLimit, wei)
    fmt.Println("\nRaw TX:\n", rawTxRLPHex)

    sendRawTransaction(rawTxRLPHex, sepoliaRpcUrl) // This was just added.
}
```

Run with **`go run .`**

![Image of Raw Transaction Receipt using EthClient](https://static.wixstatic.com/media/706568_8ec51bf2212d4d85a4723fafd62966b4~mv2.png/v1/fill/w_592,h_399,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_8ec51bf2212d4d85a4723fafd62966b4~mv2.png)

We can see the transaction receipt in the image above

### Signing Ethereum messages (digital signature)

Ethereum-signed messages can be used to create verification systems. It is a way to verify ownership or consent without performing an on-chain transaction.

For example, if User A signs a message with their private key and submits it to a platform, the platform takes the user's public address, the message, and the signature and verifies if the signature was indeed signed by User A; if yes, it could serve as an authorization for the platform to do something (whatever the reason was for signing).

Ethereum message signing utilizes the secp256k1 elliptic curve digital signature algorithm (ECDSA) for cryptographic security.

Ethereum-signed messages also have a prefix, so they are recognizable and unique to the network.

The prefix is: `\x19Ethereum Signed Message:\n" + len(message)`, and then we hash the `prefix+message` before signing it: `sign(keccak256("\x19Ethereum Signed Message:\n" + len(message) + message)))`.

Ethereum also has a recovery ID that is added to the last byte of the signature. The signature is 65 bytes long, split into 3 parts: v, r, and s. r is the first 32 bytes, s is the next 32 bytes, and v is one byte representing the recovery ID.

The recovery ID is either 27 `(0x1b)` or 28 `(0x1c)` for Ethereum. You'd usually see this at the end of all Ethereum digital signatures (or signed messages).

The **crypto** package from Geth used for signing does not add the recovery ID like how Metamask `personal_sign` does, so we have to manually add this after signing with `sig[64]+=27` as you'd see in the code below.

Note that `signing messages is completely done off-chain and offline`. It doesn't make any JSON-RPC call.

Add the code below a 'signMessage.go' file, in the project directory.

```go!
package mainimport (
    "crypto/ecdsa"
    "encoding/json"
    "fmt"
    "log"
    "github.com/ethereum/go-ethereum/common/hexutil"
    "github.com/ethereum/go-ethereum/crypto"
)
    // SignatureResponse represents the structure of the signature response.
type SignatureResponse struct {
    Address string `json:"address,omitempty"`
    Msg     string `json:"msg,omitempty"`
    Sig     string `json:"sig,omitempty"`
    Version string `json:"version,omitempty"`
}

// signMessage signs a message using the provided private key.
func signMessage(message, privKey string) (string, string) {
    // Convert the private key from hex to ECDSA format
    ecdsaPrivateKey, err := crypto.HexToECDSA(privKey)
    if err != nil {
        log.Fatalln(err)
    }

    // Construct the message prefix
    prefix := []byte(fmt.Sprintf("\x19Ethereum Signed Message:\n%d", len(message)))    messageBytes := []byte(message)

    // Hash the prefix and message using Keccak-256
    hash := crypto.Keccak256Hash(prefix, messageBytes)

    // Sign the hashed message
    sig, err := crypto.Sign(hash.Bytes(), ecdsaPrivateKey)
    if err != nil {
        log.Fatalln(err)
    }

    // Adjust signature ID to Ethereum's format
    sig[64] += 27

    // Derive the public key from the private key
    publicKeyBytes := crypto.FromECDSAPub(ecdsaPrivateKey.Public().(*ecdsa.PublicKey))
    pub, err := crypto.UnmarshalPubkey(publicKeyBytes)
    if err != nil {
        log.Fatal(err)
    }
    rAddress := crypto.PubkeyToAddress(*pub)

    // Construct the signature response
    res := SignatureResponse{
        Address: rAddress.String(),
        Msg:     message,
        Sig:     hexutil.Encode(sig),
        Version: "2",    }

    // Marshal the response to JSON with proper formatting
    resBytes, err := json.MarshalIndent(res, " ", "\t")
    if err != nil {
        log.Fatalln(err)
    }

    return res.Sig, string(resBytes)
}
```

Again, update the main function in main.go:

```go!
func main() {
    fmt.Println("using ethclient...")

    getSuggestedGasPrice(sepoliaRpcUrl)

    eGas := estimateGas(sepoliaRpcUrl, from, to, data, wei)
    fmt.Println("\nestimate gas for the transaction is:", eGas)

    rawTxRLPHex := createRawTransaction(sepoliaRpcUrl, to, data, privKey, gasLimit, wei)
    fmt.Println("\nRaw TX:\n", rawTxRLPHex)

    sendRawTransaction(rawTxRLPHex, sepoliaRpcUrl)

    sig, sDetails := signMessage(data, privKey) // This was just added.
    fmt.Println("\nsigned message:", sDetails) // This was just added.
}
```

You should get this when you run the code:

![Terminal output of verifying a signature using ethclient](https://static.wixstatic.com/media/706568_3f7f8a65423340bba6b47c481cddf2c6~mv2.png/v1/fill/w_592,h_66,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_3f7f8a65423340bba6b47c481cddf2c6~mv2.png)

### 5. Verifying signatures of signed Ethereum messages

As mentioned in the last section, we can sign and verify signed messages offline. To verify a signed message, we need the signature, the address of the signer, and the original message.

The `verifySig` function below takes these parameters, decodes the signature into bytes, and removes the Ethereum recovery ID. The reason for this is because the `crypto` package used for signing and verifying signatures checks that the recovery ID (65th byte) of the signature is less than 4 (my guess is that so it's not limited to just Ethereum signatures).

After this, we reconstruct the necessary parameters (details in the code below) and call the `crypto.Ecrecover` function, which works similarly to the [EVM Ecrecover precompile contract at address (0x01)](https://www.rareskills.io/post/solidity-precompiles#:~:text=Address%200x01%3A%20ecRecover), which returns the address that signed the message (created the signature).

Create a `verifySignedMessage.go` file in the project and add this code:

```go!
package main

import (
    "fmt"
    "log"    "strings"
    "github.com/ethereum/go-ethereum/common/hexutil"
    "github.com/ethereum/go-ethereum/crypto"
)

// handleVerifySig verifies the signature against the provided public key and hash.
func verifySig(signature, address, message string) bool {
    // Decode the signature into bytes
    sig, err := hexutil.Decode(signature)
    if err != nil {
        log.Fatalln(err)
    }

    // Adjust signature to standard format (remove Ethereum's recovery ID)
    sig[64] = sig[64] - 27

    // Construct the message prefix
    prefix := []byte(fmt.Sprintf("\x19Ethereum Signed Message:\n%d", len(message)))
    data := []byte(message)

    // Hash the prefix and data using Keccak-256
    hash := crypto.Keccak256Hash(prefix, data)

    // Recover the public key bytes from the signature
    sigPublicKeyBytes, err := crypto.Ecrecover(hash.Bytes(), sig)
    if err != nil {
        log.Fatalln(err)
    }
    ecdsaPublicKey, err := crypto.UnmarshalPubkey(sigPublicKeyBytes)
    if err != nil {
        log.Fatalln(err)
    }

    // Derive the address from the recovered public key
    rAddress := crypto.PubkeyToAddress(*ecdsaPublicKey)

    // Check if the recovered address matches the provided address
    isSigner := strings.EqualFold(rAddress.String(), address)

    return isSigner
}
```

Update the main function in main.go:

```go!
func main() {
    fmt.Println("using ethclient...")

    getSuggestedGasPrice(sepoliaRpcUrl)

    eGas := estimateGas(sepoliaRpcUrl, from, to, data, wei)
    fmt.Println("\nestimate gas for the transaction is:", eGas)

    rawTxRLPHex := createRawTransaction(sepoliaRpcUrl, to, data, privKey, gasLimit, wei)
    fmt.Println("\nRaw TX:\n", rawTxRLPHex)

    sendRawTransaction(rawTxRLPHex, sepoliaRpcUrl)

    sig, sDetails := signMessage(data, privKey)
    fmt.Println("\nsigned message:", sDetails)

    if isSigner := verifySig(sig, from, data); isSigner { // This was just added.
        fmt.Printf("\n%s signed %s\n", from, data)
    } else {
        fmt.Printf("\n%s did not sign %s\n", from, data)
    }
}
```

We can now confirm if the private key signed the message, which it did. Run go run .:

![Terminal output of a signed message by a private key using verifySig ](https://static.wixstatic.com/media/706568_51ab86f01b4b4e999bf6f72449392de2~mv2.png/v1/fill/w_592,h_30,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_51ab86f01b4b4e999bf6f72449392de2~mv2.png)

**Exercise**: Pass a different message to the verifySig function. You should get: `0x571B102323C3b8B8Afb30619Ac1d36d85359fb84 did not sign Hello Ethereum!`, because of incorrect data.

### 6. Retrieving an account's nonce (number of transaction)

To get the nonce of an account, we can either use the "PendingNonceAt" or the "NonceAt" function. PendingNonceAt returns the next unused nonce for the account, while NonceAt returns the current nonce for the account.

Another difference is that PendingNonceAt just gets the next nonce, while NonceAt tries to get the nonce of the account on a specified block number; if non is passed, it returns the nonce of the account on the last known block.

Both methods initiate JSON-RPC calls using `eth_getTransactionCount`; however, the first includes a second parameter of "pending," while the other specifies the block number.

Now, create a 'getNonce.go' file and paste the code below:

```go!
package main

import (
    "context"
    "fmt"
    "log"
    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/ethclient"
)

// getNonce fetches and prints the current and next nonce for a given Ethereum address.
func getNonce(address, rpcUrl string) (uint64, uint64) {
    client, err := ethclient.Dial(rpcUrl)
    if err != nil {
        log.Fatalln(err)
    }

    // Retrieve the next nonce for the address
    nextNonce, err := client.PendingNonceAt(context.Background(), common.HexToAddress(address))
    if err != nil {
        log.Fatalln(err)
    }

    var currentNonce uint64 // Variable to hold the current nonce.
    if nextNonce > 0 {
        currentNonce = nextNonce - 1
    }

    return currentNonce, nextNonce
}
```

Update the main function:

```go!
func main() {
    fmt.Println("using ethclient...")

    getSuggestedGasPrice(sepoliaRpcUrl)

    eGas := estimateGas(sepoliaRpcUrl, from, to, data, wei)
    fmt.Println("\nestimate gas for the transaction is:", eGas)

    rawTxRLPHex := createRawTransaction(sepoliaRpcUrl, to, data, privKey, gasLimit, wei)
    fmt.Println("\nRaw TX:\n", rawTxRLPHex)

    sendRawTransaction(rawTxRLPHex, sepoliaRpcUrl)

    sig, sDetails := signMessage(data, privKey)
    fmt.Println("\nsigned message:", sDetails)

    if isSigner := verifySig(sig, from, data); isSigner {
        fmt.Printf("\n%s signed %s\n", from, data)
    } else {
        fmt.Printf("\n%s did not sign %s\n", from, data)
    }

    cNonce, nNonce := getNonce(to, sepoliaRpcUrl) // This was just added.
    fmt.Printf("\n%s current nonce: %v\n", to, cNonce) // This was just added.
    fmt.Printf("%s next nonce: %v\n", to, nNonce) // This was just added.
}
```

`go run .` the program, you should see this:

![Terminal output image of an account's nonce using eth_getTransactionCount](https://static.wixstatic.com/media/706568_2aa402d39e5b40919fdecb88eada67ee~mv2.png/v1/fill/w_592,h_54,al_c,lg_1,q_85,enc_auto/706568_2aa402d39e5b40919fdecb88eada67ee~mv2.png)


### 7. Tracing a Transaction

As mentioned earlier, we'll be using Geth's 'rpc' package for transaction tracing, a functionality not directly supported by ethclient.

By tracing transactions, we can visualize the execution path and gain insights into any event logs during execution of the transaction.

For this, we will focus on two main methods: `debug_traceTransaction` and a custom RPC method by otterscan `ots_traceTransaction` (explained below).

`debug_traceTransaction` uses Geth's native transaction tracing, which takes in the transaction hash and a trace configuration, specifying the type of trace to do. Geth has different native tracers, but we will be using the "callTracer". To see all available Geth native tracers, you can read the [**documentation**](https://geth.ethereum.org/docs/developers/evm-tracing/built-in-tracers#native-tracers) later.

`debug_traceTransaction` leverages Geth's built-in transaction tracing capabilities. It requires two arguments:

-   The transaction hash, and

-   A trace configuration: This specifies the details of the trace, such as the type of information to capture. Geth offers various native tracers, but for this example, we'll focus on the "callTracer". This tracer tracks all the call frames (function call) executed during a transaction execution.

An example of a trace generated using the 'callTracer' configuration:

```go!
client.CallContext(
    context.Background(),
    &result,
    "debug_traceTransaction",
"0xd12e31c3274ff32d5a73cc59e8deacbb0f7ac4c095385add3caa2c52d01164c1",
    map[string]any{
        "tracer": "callTracer",
        "tracerConfig": map[string]any{"withLog": true}}
)
```

The config parameter, following the transaction hash, instructs the connected Geth node to perform a call trace and include any generated event logs.

We will use o`ts_traceTransaction` (see explanation after the code).

Create a `traceTx.go` file in our project and paste the code below:

```go!
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "github.com/ethereum/go-ethereum/rpc"
)

func traceTx(hash, rpcUrl string) string {
    var (
        client *rpc.Client // Define a variable to hold the RPC client.
        err    error       // Variable to catch errors.
    )

    // Connect to the Ethereum RPC endpoint using the provided URL.
    client, err = rpc.Dial(rpcUrl)
    if err != nil {
        log.Fatalln(err)
    }

    var result json.RawMessage // Variable to hold the raw JSON result of the call.

    // Make the RPC call to trace the transaction using its hash. `ots_traceTransaction` is the method name.
    err = client.CallContext(context.Background(), &result, "ots_traceTransaction", hash) // or use debug_traceTransaction with a supported RPC URL and params: hash, map[string]any{"tracer": "callTracer", "tracerConfig": map[string]any{"withLog": true}} for Geth tracing
    if err != nil {
        log.Fatalln(err)
    }

    // Marshal the result into a formatted JSON string
    resBytes, err := json.MarshalIndent(result, " ", "\t")
    if err != nil {
        log.Fatalln(err)
    }

    return string(resBytes))
}
```

Update the main function:

```go!
func main() {
    fmt.Println("using ethclient...")

    getSuggestedGasPrice(sepoliaRpcUrl)

    eGas := estimateGas(sepoliaRpcUrl, from, to, data, wei)
    fmt.Println("\nestimate gas for the transaction is:", eGas)

    rawTxRLPHex := createRawTransaction(sepoliaRpcUrl, to, data, privKey, gasLimit, wei)
    fmt.Println("\nRaw TX:\n", rawTxRLPHex)

    sendRawTransaction(rawTxRLPHex, sepoliaRpcUrl)

    sig, sDetails := signMessage(data, privKey)
    fmt.Println("\nsigned message:", sDetails)

    if isSigner := verifySig(sig, from, data); isSigner {
        fmt.Printf("\n%s signed %s\n", from, data)    }
    else {
        fmt.Printf("\n%s did not sign %s\n", from, data)
    }

    cNonce, nNonce := getNonce(to, sepoliaRpcUrl)
    fmt.Printf("\n%s current nonce: %v\n", to, cNonce)
    fmt.Printf("%s next nonce: %v\n", to, nNonce)

    res := traceTx("0xd12e31c3274ff32d5a73cc59e8deacbb0f7ac4c095385add3caa2c52d01164c1", mainnetRpcUrl) // This was just added.
    fmt.Println("\ntrace result:\n", res) // This was just added.

}
```

`ots_traceTransaction` is a custom Ethereum JSON-RPC method for transaction tracing developed by [**Otterscan**](https://github.com/otterscan/otterscan) and it is not part of Geth. It only requires the transaction hash as input and returns a structured trace output without any logs.

Note that the Sepolia RPC URL in the sepoliaRpcUrl variable does not support the `ots_traceTransaction` method. For this example, we'll use the mainnet RPC URL stored in the mainnetRpcUrl variable, which does support it.

After running the program, we should see the call trace.

![Image of trace result using ots_traceTransaction](https://static.wixstatic.com/media/706568_ecf5e50e8e2f4c4a80d3132e4d923045~mv2.png/v1/fill/w_592,h_585,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_ecf5e50e8e2f4c4a80d3132e4d923045~mv2.png)

**Exercise**: Modify the traceTx function to use Geth's `debug_traceTransaction` with the previously demonstrated callTracer config. Use the sepoliaRpcUrl and a corresponding Sepolia transaction hash for the trace.

You should see a slightly different trace output than the previous one that looks like this:

![Image of Trace Result using Geth's debug_traceTransaction](https://static.wixstatic.com/media/706568_37fa1f5c1c3343aa87a69220ebb45a18~mv2.png/v1/fill/w_592,h_446,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_37fa1f5c1c3343aa87a69220ebb45a18~mv2.png)

### 8. Create and send EIP-4844 blob transaction

The dencun hard-fork just went live on Ethereum and it introduced several EIPs, with one being EIP-4844 with a new transaction type called blob transaction (type 3).

Blobs are short form for **B**inary **L**arge **Ob**ject**s**. In the case of Ethereum, it is a transaction data that persist on the consensus layer of ethereum and not the execution layer like other transactions. Therefore, you need consensus clients like Prysm to access them and not Geth which is an execution client.

#### **Blob transactions field**

Blob transaction has similar fields to EIP-1559 transaction but with some added fields like **blob_versioned_hashes** (vector of sha256 hashes), and **max_fee_per_blob_gas**(uint256) and a rule that the transaction **to** field must not be nil.

The versioned hash of a blob which is 32 bytes, is made up of a single byte representing the version (currently 0x01, will likely change when Ethereum moves to full-sharding) at the start, followed by the last 31 bytes of the SHA256 hash of the KZG commitment of the blob(explained below).

**versioned hash**

```go!
/ go-ethereum/crypto/kzg4844/kzg4844.go
// CalcBlobHashV1 calculates the 'versioned blob hash' of a commitment.

func CalcBlobHashV1(hasher hash.Hash, commit *Commitment) (vh [32]byte) {
    if hasher.Size() != 32 {
    panic("wrong hash size")
    }

    hasher.Reset()
    hasher.Write(commit[:])
    hasher.Sum(vh[:0]) // save the commitment hash to `vh`
    vh[0] = 0x01 // set hash version

    return vh
}
```

#### **Where are blob stored?**

The full contents of a blob are not embedded to a block nor persisted on the execution layer and is not accessible in the EVM, instead are managed seperately by the beacon chain (consensus layer) as blob sidecars in other to save block space for normal transaction execution.

A Sidecar can contain one or a list of blobs (128 bytes each), a list of their corresponding kzg commitment (48 bytes each) and a list of their corresponding kzg proof (48 bytes each).

Blobs are stored in the beacon chain for 18 days and then they're pruned after. Rollups can deal with this expiry by also storing blobs theirselves or use p2p storage to store blobs.

**Blob sidecar**

```go!
// go-ethereum/core/types/tx_blob.go
// BlobTxSidecar contains the blobs of a blob transaction.
type BlobTxSidecar struct {
    Blobs       []kzg4844.Blob       // Blobs
    Commitments []kzg4844.Commitment // Blob commitments
    Proofs      []kzg4844.Proof      // Blob KZG proofs
}
```

The blob is used to compute the KZG commitment, and the blob togther with the KZG commitment is used to compute the KZG proof. This proof is used to verify the blob against the commitment.

#### **What are blobs used for?**

The main use case for blobs are to handle layer 2s and rollups block data, instead of using calldata which users also use, leading to a competion for block space. Using a seperate transaction (blobs) reduce cost for layer 2s and rollups.

**However, blobs are not limited to just rollups and can be used by anyone**. I will demonstrate how to send a blob transaction later.

#### **Limit of blob per transaction**

We can send more than one blob per transaction, however, there is a target of 3 and maximum of 6 blobs per block limit, so technically a blob transaction can contain up to six blobs if it were the only blob transaction in the block. This also implies that the sidecar which contains these blobs will contain the same amount of blob commitment and versioned hashes as the number of blobs in it, six in this case.

#### **Blob gas**

Blob transactions use a different type of gas called blob gas and have the following parameters: **MAX_BLOB_GAS_PER_BLOCK** of 786,432; **TARGET_BLOB_GAS_PER_BLOCK** of 393,216; and **MIN_BLOB_BASE_FEE** of 1.

It is seperate from the existing transaction gas we know. The blob gas fee has a similar pricing mechanism to EIP-1559 in that it increases and decreases based on network congestion. It increases if the previous block uses more gas than the **TARGET_BLOB_GAS_PER_BLOCK** (~3 blobs) and decreases when the previous block uses less.

Note that the blob versioned hashes are stored as references to blobs in the execution layer. But the blobs are not stored in the execution layer. Blobs do not need priority fee because the blob data will not be executed.

Finally, creating a blob transaction in Go follows a very similar step to a normal transaction, except that we use the `types.BlobTx` struct and pass the blob related fields as hinted earlier.



Create a `blobTx.go` file and paste the following code:

```go!
package transaction

import (
    "context"
    "fmt"
    "regexp"
    "strings"
    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/common/hexutil"
    "github.com/ethereum/go-ethereum/core/types"
    "github.com/ethereum/go-ethereum/crypto"
    "github.com/ethereum/go-ethereum/crypto/kzg4844"=
    "github.com/ethereum/go-ethereum/ethclient"
    "github.com/holiman/uint256"
)

// SendBlobTX sends a transaction with an EIP-4844 blob payload to the Ethereum network.

func sendBlobTX(rpcURL, toAddress, data, privKey string) (string, error) {
    // Connect to the Ethereum client
    client, err := ethclient.Dial(rpcURL)
    if err != nil {
        return "", fmt.Errorf("failed to dial RPC client: %s", err)
    }
    defer client.Close() // Ensure the connection is closed after completing the function

    // Retrieve the current chain ID
    chainID, err := client.ChainID(context.Background())
    if err != nil {
        return "", fmt.Errorf("failed to get chain ID: %s", err)
    }

    var Blob [131072]byte // Define a blob array to hold the large data payload, blobs are 128kb in length

    // If necessary, convert the input data to a byte slice in hex format
    var bytesData []byte
    if data != "" {
        // Check if the data is in hex format, with or without the '0x' prefix
        if IsHexWithOrWithout0xPrefix(data) {
            // Ensure the data has the '0x' prefix
             if !strings.HasPrefix(data, "0x") {
                data = "0x" + data
            }
        // Decode the hex-encoded data
        bytesData, err = hexutil.Decode(data)
        if err != nil {
            return "", fmt.Errorf("failed to decode data: %s", err)
        }
        // Copy the decoded data into the blob array
        copy(Blob[:], bytesData)
        } else {
            // If the data is not in hex format, copy it directly into the blob array
            copy(Blob[:], data)
        }
    }

    // Compute the commitment for the blob data using KZG4844 cryptographic algorithm
    BlobCommitment, err := kzg4844.BlobToCommitment(Blob)
    if err != nil {
        return "", fmt.Errorf("failed to compute blob commitment: %s", err)
    }

    // Compute the proof for the blob data, which will be used to verify the transaction
    BlobProof, err := kzg4844.ComputeBlobProof(Blob, BlobCommitment)
    if err != nil {
        return "", fmt.Errorf("failed to compute blob proof: %s", err)
    }

    // Prepare the sidecar data for the transaction, which includes the blob and its cryptographic proof
    sidecar := types.BlobTxSidecar{
        Blobs: []kzg4844.Blob{Blob},
        Commitments: []kzg4844.Commitment{BlobCommitment},
    Proofs: []kzg4844.Proof{BlobProof},
    }

    // Decode the sender's private key
    pKeyBytes, err := hexutil.Decode("0x" + privKey)
    if err != nil {
        return "", fmt.Errorf("failed to decode private key: %s", err)
    }

    // Convert the private key into the ECDSA format
    ecdsaPrivateKey, err := crypto.ToECDSA(pKeyBytes)
    if err != nil {
        return "", fmt.Errorf("failed to convert private key to ECDSA: %s", err)
    }

    // Compute the sender's address from the public key
    fromAddress := crypto.PubkeyToAddress(ecdsaPrivateKey.PublicKey)

    // Retrieve the nonce for the transaction
    nonce, err := client.PendingNonceAt(context.Background(), fromAddress)
    fmt.Println(nonce)

    if err != nil {
        return "", fmt.Errorf("failed to get nonce: %s", err)
    }

    // Create the transaction with the blob data and cryptographic proofs
    tx, err := types.NewTx(&types.BlobTx{
        ChainID: uint256.MustFromBig(chainID),
        Nonce: nonce,
        GasTipCap: uint256.NewInt(1e10), // max priority fee per gas
        GasFeeCap: uint256.NewInt(50e10), // max fee per gas
        Gas: 250000, // gas limit for the transaction
        To: common.HexToAddress(toAddress), // recipient's address
        Value: uint256.NewInt(0), // value transferred in the transaction
        Data: nil, // No additional data is sent in this transaction
        BlobFeeCap: uint256.NewInt(3e10), // fee cap for the blob data
        BlobHashes: sidecar.BlobHashes(), // blob hashes in the transaction
        Sidecar: &sidecar, // sidecar data in the transaction
    }), err

    if err != nil {
        return "", fmt.Errorf("failed to create transaction: %s", err)
    }

    // Sign the transaction with the sender's private key
    signedTx, err := types.SignTx(tx, types.LatestSignerForChainID(chainID), ecdsaPrivateKey)

    if err != nil {
    return "", fmt.Errorf("failed to sign transaction: %s", err)
    }

    // Send the signed transaction to the Ethereum network
    if err = client.SendTransaction(context.Background(), signedTx); err != nil {
        return "", fmt.Errorf("failed to send transaction: %s", err)
    }

    // Return the transaction hash
    txHash := signedTx.Hash().Hex()

    return txHash, nil
}

// IsHexWithOrWithout0xPrefix checks if a string is hex with or without `0x` prefix using regular expression.
func IsHexWithOrWithout0xPrefix(data string) bool {
    pattern := `^(0x)?[0-9a-fA-F]+$`
    matched, _ := regexp.MatchString(pattern, data)
    return matched
}
```

Update the main function:

```go!
func main() {
    fmt.Println("using ethclient...")

    getSuggestedGasPrice(sepoliaRpcUrl)

    eGas := estimateGas(sepoliaRpcUrl, from, to, data, wei)
    fmt.Println("\nestimate gas for the transaction is:", eGas)

    rawTxRLPHex := createRawTransaction(sepoliaRpcUrl, to, data, privKey, gasLimit, wei)
    fmt.Println("\nRaw TX:\n", rawTxRLPHex)     sendRawTransaction(rawTxRLPHex, sepoliaRpcUrl)

    sig, sDetails := signMessage(data, privKey)
    fmt.Println("\nsigned message:", sDetails)

    if isSigner := verifySig(sig, from, data); isSigner {
        fmt.Printf("\n%s signed %s\n", from, data)
    } else {
        fmt.Printf("\n%s did not sign %s\n", from, data)
    }

    cNonce, nNonce := getNonce(to, sepoliaRpcUrl)
    fmt.Printf("\n%s current nonce: %v\n", to, cNonce)
    fmt.Printf("%s next nonce: %v\n", to, nNonce)

    res := traceTx("0xd12e31c3274ff32d5a73cc59e8deacbb0f7ac4c095385add3caa2c52d01164c1", mainnetRpcUrl)
    fmt.Println("\ntrace result:\n", res)

    blob, err := sendBlobTX(sepoliaRpcUrl, to, data, privKey) // This was just added.
    if err != nil {
        log.Fatalln(err)
    }

    fmt.Println("\nBlob transaction hash:", blob) // This was just added.
}
```

Before running our program, temporarily comment out the sendRawTransaction and traceTx function calls. This is because a pending transaction from sendRawTransaction can cause a nonce conflict (nonce gap error) when creating the blob transaction, and subsequent traceTx call will clutter the terminal output.

After doing that, run with **`go run .`**. You should get the transaction hash.

![Terminal Image of a Blob Transaction Hash](https://static.wixstatic.com/media/706568_4f6954ef99784ca581955c5daae32d76~mv2.png/v1/fill/w_592,h_34,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_4f6954ef99784ca581955c5daae32d76~mv2.png)

You can look it up on Etherscan, here is the one I made:

[https://sepolia.etherscan.io/blob/0x0142681987b40afb99da6ab299794cd4ab4304c92bec12d2f375c0e52dbd7e9b?bid=481872](https://sepolia.etherscan.io/blob/0x0142681987b40afb99da6ab299794cd4ab4304c92bec12d2f375c0e52dbd7e9b?bid=481872)

## Summary

The Go-Ethereum (Geth) `ethclient` package simplifies many common interactions with Ethereum. However, like every other client out there, it doesn't provide methods for all Ethereum JSON-RPC APIs, as we have seen with transaction tracing. In cases like this, manually constructing a JSON-RPC call is necessary. Fortunately, the Geth `rpc` package makes this easier for Go developers.

*Originally Published April 3, 2024*

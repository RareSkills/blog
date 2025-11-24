# Web3.js Example. Latest version 4.x. Transfer, Mint and Query the Blockchain

The newest version of web3.js, 4.x, has just been unveiled. In this guide, we’ll delve into integrating web3.js into HTML to Transfer, Mint NFTs as well as querying the Blockchain. Several features within the web3.js v 1.10.0 documentation are set to be deprecated, therefore we have compiled the most up-to-date methods for this web3.js tutorial. We will refrain from using a framework like React. Without using any frameworks, we are compelled to acquire a deeper understanding of the fundamental concepts involved in Dapp development. This hands-on approach helps you build a robust foundation for your blockchain development skills. Simply follow the step-by-step instructions, copy-paste codes, and read through the provided code snippets. Here’s what we cover in this tutorial:

-   Web3.js 4.x Migration Highlights
-   Getting Started: Web3.js
-   Part 1: Connect to Metamask
-   Part 2: Display Account Information
-   Part 2: Send Transaction
-   Part 3: Interact with a Smart Contract

We provide examples and visuals throughout this tutorial to illustrate the concepts being discussed.

## Web3.js 4.x Migration Highlights

This guide walks you through some of the significant code changes for the migration to Web3.js 4.x.

### **Breaking Changes**

#### 1. Module Import

```js
// Web3.js 1.x import
const Web3 = require('web3');

// Web3.js 4.x import
const { Web3 } = require('web3');
```

In Web3.js 4.x, we’ve switched to the destructuring syntax for importing the Web3 object.

#### 2. Default Providers

In Web3.js 4.x, the **web3.givenProvider** and **web3.currentProvider** defaults are **undefined**. Previously, they were **null** when web3 was instantiated without a provider.

```js
// Defaults to undefined in Web3.js 4.x, previously null
console.log(web3.givenProvider);
console.log(web3.currentProvider);
```

If you’re using MetaMask, it’s recommended to pass **window.ethereum** directly to the Web3 constructor, instead of using **Web3.givenProvider**.

```js
// Instantiate Web3 with MetaMask providerconst web3 = new Web3(window.ethereum);
```

#### **3. Callbacks in Functions**

Callbacks are no longer supported in functions, except for event listeners. Here’s an example of invalid use of callbacks in Web3.js 4.x, which was valid in version 1:

```js
// INVALID in Web3.js 4.x
web3.eth.getBalance("0x407d73d8a49eeb85d32cf465507dd71d507100c1", function(err, result) {
        if (err) {
                console.log(err);
        } else {
        console.log(result);
        }
});
```

### **Minor Changes**

#### **1. Instantiating a Contract**

In Web3.js 4.x, you must use the **new** keyword to instantiate a **web3.eth.Contract()** object.

```js
// Web3.js 1.x
const contract = web3.eth.Contract(jsonInterface, address);

// Web3.js 4.x
const contract = new web3.eth.Contract(jsonInterface, address);
```

#### **2. getBalance**

In Web3.js 4.x, **getBalance** returns a BigInt instead of a String. For the full migration guide visit the **web3.js docs**: [https://web3js.org/#/](https://web3js.org/#/)

## Getting Started: Web3.js

Create a folder with the following directory:

```
├── directory/
│   ├── index.html
│   ├── main.js
│   ├── styles.css
```

Where to download web3.js:

-   **CDN** link: web3.js


Find this project on:

-   **GitHub** project Link: https://github.com/AymericRT/web3.js.git

## Part 1: Connect to MetaMask

We will now create a button that establishes a connection with MetaMask. The frontend codes will be put in **index.html**and the web3.js JavaScript codes in **main.js**

### **Index.html**

The codes below contain the following:

-   HTML boilerplate
-   MetaMask button
-   web3.js CDN package Script Tag
-   main.js Script Tag

```html!
<!DOCTYPE html>
<html lang="en">
      <head>
            <meta charset="UTF-8" />
            <meta http-equiv="X-UA-Compatible" content="IE=edge" />
            <meta name="viewport" content="width=device-width, initial-scale=1.0" />
            <title>Web3.js Example</title>
            <link rel="stylesheet" href="./styles.css" />
            <script
                  src="https://cdnjs.cloudflare.com/ajax/libs/web3/4.0.1-alpha.5/web3.min.js"
                  integrity="sha512-NfffVWpEqu1nq+658gzlJQilRBOvjs+PhEKSjT9gkQXRy9OfxI2TOXr33zS7MgGyTpBa5qtC6mKJkQGd9dN/rw=="
                  crossorigin="anonymous"
                  referrerpolicy="no-referrer"
            ></script>
      </head>
      <body>
            <main>
                  <p id="status1" class="status" style="color: red">disconnected</p>
                  <p id="status2" style="color: white"></p>
                  <div class="maincontainer">

                        <!-- Connect Wallet -->
                        <div class="container">
                              <div class="buttonswrapper">
                                    <div class="buttonswrapperGrid">
                                          <button id="metamask" class="button28">MetaMask</button>
                                    </div>
                              </div>
                        </div>

                        <!-- Account Info Button -->

                        <!-- Send Transaction -->

                        <!-- Mint -->

                  </div>
            </main>
            <script src="./main.js"></script>
      </body>
</html>
```

### **HTML Code snippet elaborations**

```html!
<script src="https://cdnjs.cloudflare.com/ajax/libs/web3/4.0.1-alpha.5/web3.min.js"
    integrity="sha512-NfffVWpEqu1nq+658gzlJQilRBOvjs+PhEKSjT9gkQXRy9OfxI2TOXr33zS7MgGyTpBa5qtC6mKJkQGd9dN/rw=="
    crossorigin="anonymous" referrerpolicy="no-referrer">
```

This script tag in the <head> section is used to load the web3.js CDN library.

This is the equivalent of **const { Web3 } = require(‘web3’);**

```html
<script src="./main.js"></script>
```

This loads our “main.js” functions into the HTML. Best practice is to place it at bottom of body to avoid accessing DOM elements before rendering.

### **Styles.css**

Copy paste the css codes from GitHub: GitHub link to file: [https://github.com/AymericRT/web3.js/blob/master/styles.css](https://github.com/AymericRT/web3.js/blob/master/styles.css)

### Main.js File

In the main.js file, we will incorporate the necessary JavaScript functions to activate the MetaMask button. There are three primary functions below:

-   **Event Listener for “metamask” Button**
    -   Upon a click event, it checks if MetaMask is available and connected.
    -   If yes, it calls the **ConnectWallet** function.
    -   Otherwise, it logs an error message to the console and updates the status on the web page to indicate the absence of MetaMask.
-   **checkMetaMaskAvailability()**
    -   The function is used to determine if a MetaMask browser extension is present and connected.
    -   It checks for the existence of **window.ethereum** and attempts to request access to MetaMask accounts through **ConnectMetaMask()**.
    -   If the account access is granted, the function returns **true**, indicating successful connection. Otherwise, it logs an error message and returns **false**.
-   **ConnectMetaMask();**
    -   This function is responsible for connecting to MetaMask.
    -   It attempts to request access to MetaMask accounts using **eth_requestAccounts.**

Cross-reference the explanations above with the codes below.

```js
const web3 = new Web3(window.ethereum);

// Function to check if MetaMask is available
async function checkMetaMaskAvailability() {
    if (window.ethereum) {
        try {
            // Request access to MetaMask accounts
            await window.ethereum.request({ method: "eth_requestAccounts" });
            return true;
        } catch (err) {
            console.error("Failed to connect to MetaMask:", err);
            return false;
        }
    } else {
        console.error("MetaMask not found");
        return false;
    }
}

// Event listener for MetaMask button
document.getElementById("metamask").addEventListener("click", async () => {
    const metaMaskAvailable = await checkMetaMaskAvailability();
    if (metaMaskAvailable) {
        await ConnectWallet();
    } else {
        // MetaMask not available
        console.error("MetaMask not found");
        // Update status
        document.getElementById("status1").innerText = "MetaMask not found";
        document.getElementById("status1").style.color = "red";
    }
});

//Function to connect to MetaMask
async function ConnectWallet() {
    try {
        // Request access to MetaMask accounts
        await window.ethereum.request({ method: "eth_requestAccounts" });
        // Update status
        document.getElementById("status1").innerText = "Connected to MetaMask";
        document.getElementById("status1").style.color = "green";
    } catch (err) {
        // Handle error
        console.error("Failed to connect to MetaMask:", err);
        // Update status
        document.getElementById("status1").innerText = "Failed to connect to MetaMask";
        document.getElementById("status1").style.color = "red";
    }
}
```

### Important Code Snippets: Connect to MetaMask

#### web3.js window.ethereum

```js
const web3 = new Web3(window.ethereum);
```

**web3** is a new instance of the Web3.js library.

By assigning **window.ethereum** to the Web3 constructor, **web3** variable uses the provided Ethereum provider for interacting with the Ethereum network.

#### **web3.js request Accounts**

```js
await window.ethereum.request({ method: "eth_requestAccounts" });
```

This code requests access to the user’s Ethereum accounts in the web app using the **eth_requestAccounts** method.

### Run your server

In your terminal, navigate to your directory and run this Python command, it will automatically deploy your server in port 8000.

```bash
python -m SimpleHTTPServer 8000
```

This is how your website should look on **localhost:8000**

![Web.3js button to connect to provider, metamask](https://static.wixstatic.com/media/935a00_bad0148b520443c3a581ad99771e5408~mv2.png/v1/fill/w_666,h_353,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_bad0148b520443c3a581ad99771e5408~mv2.png)

## Part 2: Display Account Information

Now that our MetaMask is connected, we are capable of querying the blockchain for essential account details such as the connected account address, balance the current network fees. We’ll create a button for this functionality and name it “Account_Information”.

### Index.html

Insert the “account_Information” button under the comment.

```html
<!-- Account Information Button -->
<div class="secondcontainer">
    <button id="accountbutton" class="button49">
        Account Information
    </button>
</div>
```

### Main.js

We’ll write a function that fetches data from MetaMask. The data will display the following:

-   Account address
-   Account balance
-   The current network fees

The two primary functions required are:

-   **Event Listener for Account Information Button**
    -   Upon a click event, it checks if MetaMask is available and connected.
    -   If yes, it calls the **AccountInfo()** function.
    -   Otherwise, it logs an error and updates the web page status.
-   **AccountInformarion();**
    -   This function is responsible for fetching data from the Ethereum Wallet.
    -   It retrieves the account address using the **web3.eth.getAccounts()** method.
    -   The account balance is obtained through the **web3.eth.getBalance(from)** method.
    -   Additionally, it retrieves the current gas price by calling the **web3.eth.getGasPrice()** method.

```js!
// Event Listener for Account Information
document.getElementById("accountbutton").addEventListener("click", async () => {
    const metaMaskAvailable = await checkMetaMaskAvailability();
    if (metaMaskAvailable) {
        await AccountInformation();
    }
});

//Function to call the Account Information
async function AccountInformation() {
    const account = await web3.eth.getAccounts();
    const from = account[0];
    const balanceInWei = await web3.eth.getBalance(from);
    const balanceInEth = web3.utils.fromWei(balanceInWei, "ether");
    const gasPrice = await web3.eth.getGasPrice();
    const gasPriceInEth = web3.utils.fromWei(gasPrice, "ether");

    // Display the account information
    document.getElementById("status2").innerText ="Account Address: " + from + "\nBalance: " + balanceInEth + " ETH" +"\nGas Price: " + gasPriceInEth;
      document.getElementById("status2").style.color = "white";
}
```

### Important Code Snippets: Display Account Information

#### **web3.js get Accounts**

```js
const account = await web3.eth.getAccounts() //returns list of account
const from = account[0]; // gets the first account in the list
```

The **getAccounts()** method returns a list of accounts the node controls. Basically, it returns the connected MetaMask accounts.

The first element in the list will represent the primary connected account.

#### **web3.js get Balance**

```js
web3.eth.getBalance(from)
```

The **getBalance()** method fetches the balance of the account address passed to the parameter in Wei. Note that 1 Ether is equivalent to 10^18 Wei.

#### **web3.js get Gas Price**

```js
web3.eth.getGasPrice()
```

The **getGasPrice()** method retrieves the current gas price in Wei for transactions on the Ethereum network.

```js
web3.utils.fromWei(balanceInWei, "ether")
```

This method converts your balance from Wei units to Ether. If you want to convert it into GWEI, simply change “ether” to “gwei”.

![Web3.js button to retrieve account address, account balance, current gas fee](https://static.wixstatic.com/media/935a00_fa2f1094936e46859ba3a2b086b60d8f~mv2.png/v1/fill/w_666,h_295,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_fa2f1094936e46859ba3a2b086b60d8f~mv2.png)

## Part 3: Send Ether Transaction

Send Transaction requires the following arguments:

-   Sender Address
-   Recipient Address
-   Specified Amount


Our sender Address will be the default connected account. For the recipient address and the specified amount, we will generate a form with two input fields and a “send” button that will initiate the transfer.

### Index.html

Insert the form under the **<!— Send Transaction —>** comment.

```html!
<!-- Send Transaction -->
<form>
      <div class="inputcontainer">
            <input
                  id="addressinput"
                  class="myinput"
                  placeholder="Address 0x0..."
            />
            <input
                  id="amountinput"
                  class="myinput"
                  placeholder="Amount ether..."
            />
      </div>
      <div class="buttoncontainer">
            <button type="button" id="sendButton" class="button64">Send</button>
      </div>
</form>
```

### Main.js

To activate the Send Transaction Functionality, we need to create two primary functions.

-   **Event Listener for SendTransaction Button**
    -   Upon a click event, it checks if MetaMask is available and connected.
    -   If yes, it calls the **SendFunction()** function.
    -   Otherwise, it logs an error and updates the web page status.
-   **SendFunction()**
    -   This function is responsible for sending the transaction.
    -   It retrieves the recipient address and the specified amount through DOM Manipulation.
    -   Additionally creates a JavaScript object of the transaction details; from, to, amount; within **transaction**.
    -   Sends a transaction on the Ethereum network with the provided transaction details, **transaction,** through **web3.eth.sendTransaction(transaction)**

```js!
// Event Listener for Send Transaction
document.getElementById("sendButton").addEventListener("click", async () => {
    const metaMaskAvailable = await checkMetaMaskAvailability();
    if (metaMaskAvailable) {
        await SendFunction();
    }
});

//Function to call the Send Function
async function SendFunction() {
    // Get input values
    const to = document.getElementById("addressinput").value;
    const amount = document.getElementById("amountinput").value;

    // Check if both to and amount are provided
    if (!to || !amount) {
        console.error("To and amount are required");
        return;
    }

    // Convert amount to wei (1 ether = 10^18 wei)
    const amountWei = web3.utils.toWei(amount, "ether");

    // Get the selected account from MetaMask
    const accounts = await web3.eth.getAccounts();
    const from = accounts[0];

    // Create the transaction object
    const transaction = {
        from: from,
        to: to,
        value: amountWei,
    };

    // Send the transaction
    try {
        const result = await web3.eth.sendTransaction(transaction);
        console.log("Transaction result:", result);

        // Update status
        document.getElementById("status2").innerText ="Transaction sent successfully";
        document.getElementById("status2").style.color = "green";
    } catch (err) {
    // Handle error
    console.error("Failed to send transaction:", err);// Update status
    document.getElementById("status2").innerText = "Failed to send transaction";
    document.getElementById("status2").style.color = "red";
    }
}
```

### Important Code Snippets: Send Transaction

#### **web3.js Send Transaction**

```js
await web3.eth.sendTransaction(transaction)
```

**sendTransaction** takes a single parameter, transaction. Throws an error if the transaction is unsuccessful.

**transaction** is an object containing the details of the transaction to be sent. This is the format:

```js
{
    from: "Sender Address",
    to: "Recipient Address",
    value: "Amount to be sent in WEI",
};
```

Your website should look like this:

![Web3.js field form to send ether transaction](https://static.wixstatic.com/media/935a00_88fdf7cd363e40e59dc0b10d8f109361~mv2.png/v1/fill/w_666,h_617,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_88fdf7cd363e40e59dc0b10d8f109361~mv2.png)


## Part 4: Interact with Smart Contract

In this last section, you will learn how to interact with a smart contract using the web3.eth.Contract object. This includes reading data, writing data, and handling events.

-   **Reading Data:** Uses the contract’s read methods to fetch data from the contract without modifying the contract’s state and are usually free to execute.
-   **Writing Data:** Methods that modify the contract’s state or perform actions via sending transactions to the blockchain that may require gas fees.
-   **Handling Events:** Smart contracts can emit events during their execution. We can listen for these events to get notified when specific actions occur in the contract.

Here we have a contract that you will interact with:

```js
// Rareskills contractpragma solidity ^0.8.0;contract RareSkills {
  mapping(address => uint256) public balances;
  uint256 public totalSupply;

  event Mint(address indexed to, uint256 amount);

  function mint(uint256 amount) public {
    balances[msg.sender] += amount;
    totalSupply += amount;
    emit Mint(msg.sender, amount);
  }
}
```

The contract is deployed on the Mumbai Polygon network. You can check it out at the following link: [https://mumbai.polygonscan.com/](https://mumbai.polygonscan.com/)

#### Index.html

First insert the “mint” button under the comment.

```html
<!-- Minting NFT -->
<div class="mintcontainer">
    <button id="mintactual" class="button49">Mint</button>
    <p id="demo3"></p>
</div>

```

#### Main.js

In web3.js we have to first instantiate the contract in order to interact with it. There are two core elements to instantiating a contract:

-   **contract address:** 0x88d099496C1A493A36E678062f259FE9919B9150
-   **contract ABI:** Find it here

Our JavaScript will have two main functions:

-   **Event Listener for our Mint Button**
    -   Upon a click event, it checks if MetaMask is available and connected.
    -   If yes, it calls the **mintNFT()** function.
    -   Otherwise, it logs an error and updates the web page status.
-   **mintNFT()**
    -   This function is responsible for interacting with the **RareSkills contract.**
    -   Creates an instance of the contract using the **web3.eth.Contract** object, passing **contractABI** and **contractAddress** as its arguments.
    -   Retrieves the total supply of the contract by reading the **totalSupply** function.
    -   Mints the contract data by executing the **mint(uint256 amount)** function.
    -   Listens for the “**Mint** event” occurrence and handles it.

We’ll dive into each of the details in the code snippets.

```js!
// Event Listener for Mint Button
document.getElementById("mintbutton").addEventListener("click", async () => {
  const metaMaskAvailable = await checkMetaMaskAvailability();
  if (metaMaskAvailable) {
    await mintNFT();
  }
});

// Contract Details
const contractAddress = "0x88d099496C1A493A36E678062f259FE9919B9150"; // Hardcoded contract address
const contractABI = [
  // Hard coded ABI
  {
    anonymous: false,
    inputs: [
      {
        indexed: true,
        internalType: "address",
        name: "to",
        type: "address",
      },
      {
        indexed: false,
        internalType: "uint256",
        name: "amount",
        type: "uint256",
      },
    ],
    name: "Mint",
    type: "event",
  },
  {
    inputs: [
      {
        internalType: "address",
        name: "",
        type: "address",
      },
    ],
    name: "balances",
    outputs: [
      {
        internalType: "uint256",
        name: "",
        type: "uint256",
      },
    ],
    stateMutability: "view",
    type: "function",
    },
    {
      inputs: [
        {
          internalType: "uint256",
          name: "amount",
          type: "uint256",
        },],name: "mint",outputs: [],stateMutability: "nonpayable",type: "function",},{inputs: [],name: "totalSupply",outputs: [{internalType: "uint256",name: "",type: "uint256",},],stateMutability: "view",type: "function",},];

// Function to mint
async function mintNFT() {

  // Get connected account
  const accounts = await web3.eth.getAccounts();
  const from = accounts[0];

  // Instantiate a new Contract
  const contract = new web3.eth.Contract(contractABI, contractAddress);

  try {
    // Invoke contract methods
    const result = await contract.methods.mint(1).send({ from: from , value: 0});
    const _totalSupply = await contract.methods.totalSupply().call();

    document.getElementById("status2").innerText = "TotalSupply: " + _totalSupply;
    document.getElementById("status2").style.color = "green";
    document.getElementById("status3").innerText = "Minting successful";
    document.getElementById("status3").style.color = "green";

    // Event Listener
    contract
      .getPastEvents("Mint", {
        fromBlock: "latest", // Start from the latest block
        })
        .then((results) => console.log(results));

  } catch (err) {
        console.error("Failed to mint:", err);
        document.getElementById("status3").innerText = "Failed to mint";
        document.getElementById("status3").style.color = "red";
    }
}
```

## Important Code Snippets

### **web3.js Contract**

```js
const contract = new web3.eth.Contract(contractABI, contractAddress);
```

We have instantiated a new instance of RareSkills contract using the **web3.eth.Contract** constructor, passing in **contractABI** and **contractAddress** as its arguments. This instance allows us to interact directly with the smart contract located at **contractAddress** using the interface defined in **contractABI**. Interacting with the smart contract requires us to invoke the **.send()** or **.call()** function depending on whether it’s a state-viewing or a state-changing function.

**web3.js .send()**

```js
contract.methods.mint(1).send({ from: from , value: 0});
```

The **send()** method is used when executing functions that alter the state of the contract. Since our mint function adds to the totalSupply of our contract, it’s a state-changing function. The object passed to the **send()** method **{ from: from, value: 0 }** specifies the details of the transaction.

-   **from:** indicates the address from which the transaction is being sent.
-   **value:** indicates the amount of Ether that is being sent along with the transaction. In this case, no Ether is being sent.

#### **web3.js .call()**

contract.methods.totalSupply().call(); The **call()** contract method is used when executing functions that do NOT alter the contract’s state. Since the **totalSupply()** function is a “view” function, it uses the call() method.

#### **web3.js event Listeners()**

```js!
contract
      .getPastEvents("Mint", {fromBlock: "latest",}).then((results) => console.log(results));
```

The **getPastEvents** is an event handler method that is called on the **contract** object to fetch the latest event emitted.

1.  The first argument is the event name, in our case it’s the “**Mint**” event.

2.  The second argument is an options object. In this case, **fromBlock** is set to “latest” to fetch events from the latest block available. More on here.

The **results** array is logged to the console, it should look like this:

![Web3.js event handler output result example](https://static.wixstatic.com/media/935a00_b8d0eb52e02443bd864d20f13fed3986~mv2.png/v1/fill/w_666,h_315,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_b8d0eb52e02443bd864d20f13fed3986~mv2.png)

Final Website Result:

![Web3.js mint button. Interacts with smart contract](https://static.wixstatic.com/media/935a00_76aa838a2d0e4212adef5228607a65d7~mv2.png/v1/fill/w_666,h_814,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_76aa838a2d0e4212adef5228607a65d7~mv2.png)

Congratulations on finishing this web3.js tutorial!

*Originally Published May 26, 2023*

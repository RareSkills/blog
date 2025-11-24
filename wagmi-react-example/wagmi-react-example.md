# Wagmi + ReactJS Example: Transfer Crypto and Mint an NFT

In this tutorial, we'll learn how to build a [Web3](https://www.rareskills.io/web3-blockchain-bootcamps) Dapp (Decentralized Application) that connects to your crypto wallet, allowing you to transfer funds and mint NFTs. We'll be using Next.js, a popular React framework, and [Wagmi.sh](https://wagmi.sh/), a collection of React Hooks that easily integrates your wallet into your website.

Here’s an outline of the tutorial:

-   Getting Started
-   Part 1: Transferring Crypto with React + Wagmi
-   Part 2: Minting an NFT with React + Wagmi

### Authorship

This article was co-authored by Aymeric Taylor ([LinkedIn](https://www.linkedin.com/in/aymeric-russel-taylor/), [Twitter](https://twitter.com/TaylorAymeric)), a research intern at RareSkills.

## Getting Started

For the sake of simplicity, we’ll be using “Polygon Mumbai” as our test network for this tutorial, It is also supported on OpenSea.

### Connect your Wallet to Polygon Mumbai - MetaMask

Navigate through **MetaMask → Settings → Advanced** and make sure you allow **Show test networks.**

![metamask test networks](https://static.wixstatic.com/media/935a00_7f97fcc4c3d342629a84c3fdb71e5d0a~mv2.png/v1/fill/w_592,h_185,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_7f97fcc4c3d342629a84c3fdb71e5d0a~mv2.png)

Next, click on the Network Selection on the top right section and select **Add network**

![metamask add network](https://static.wixstatic.com/media/935a00_5eeaab0612f9420ea6b92ffa6021b995~mv2.png/v1/fill/w_592,h_478,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_5eeaab0612f9420ea6b92ffa6021b995~mv2.png)

If Polygon Mumbai isn’t already available to select from, you can follow the steps below.

Select **Add a network manually** and input the following:

**Network Name:** Matic Mumbai

**New RPC URL:** https://rpc-mumbai.maticvigil.com/

**Chain ID :** 80001

**Currency Symbol :** MATIC

**Block explorer URL** (optional) **:** https://mumbai.polygonscan.com/

Now simply switch onto the Polygon Network

## Wagmi Examples Part 1: Transferring Ether with React + Wagmi

### Step 1: Set up website with node js

You’ll need to have nodejs installed for this.

First create your **Next.js** project with

```bash
npx create-next-app@latest myapp
```

Check the **TypeScript** and **ESLint** option using the arrows and enter key. It should look something like this

![Create React App TypeScript](https://static.wixstatic.com/media/935a00_c0ed435320e547268cf69e749b976f5a~mv2.png/v1/fill/w_592,h_422,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_c0ed435320e547268cf69e749b976f5a~mv2.png)

Open your project in **VS Code** and install [**wagmi.sh**](https://wagmi.sh/) and **use-debounce package** (we’ll get into that later).

```bash
npm i wagmi ethers@^5
npm i use-debounce --save
```

Wagmi is basically a set of React Hooks that simplifies [Ethereum development](https://www.rareskills.io/post/ethereum-contract-creation-code) by providing useful features such as connecting wallets and interacting with contracts which we’ll learn in this tutorial. More on [wagmi.sh](https://wagmi.sh/).



### Step 2: Use configureChains to pick the network to connect to

Head over to **pages/_app.tsx** and add in the following code. Each of the functionalities is explained in the comment sections.

```tsx
import "@/styles/globals.css"; // CSS doesnt really matter now
import type { AppProps } from "next/app";
import { WagmiConfig, configureChains, createClient, mainnet } from "wagmi";
import { publicProvider } from "wagmi/providers/public";
import { polygonMumbai } from "wagmi/chains";
import { CoinbaseWalletConnector } from 'wagmi/connectors/coinbaseWallet'
import { InjectedConnector } from 'wagmi/connectors/injected'
import { MetaMaskConnector } from 'wagmi/connectors/metaMask'
import { WalletConnectConnector } from 'wagmi/connectors/walletConnect'

// configure the chains and provider that you want to use for your app,
// keep in mind that you're allowed to pass any EVM-compatible chain.
// It is also encouraged that you pass both alchemyProvider and infuraProvider.
const { chains, provider, webSocketProvider } = configureChains(
  [mainnet, polygonMumbai],
  [publicProvider()]
);

// This creates a wagmi client instance of createClient
// and passes in the provider and webSocketProvider.

const client = createClient({
  autoConnect: false,
  provider,
  webSocketProvider,
  connectors: [ // connectors is to connect your wallet, defaults to InjectedConnector();
    new MetaMaskConnector({ chains }),
    new CoinbaseWalletConnector({
      chains,
      options: {
        appName: 'wagmi',
      },
    }),
    new WalletConnectConnector({
      chains,
      options: {
        projectId: '...',
      },
    }),
    new InjectedConnector({
      chains,
      options: {
        name: 'Injected',
        shimDisconnect: true,
      },
    }),
  ],
});

export default function App({ Component, pageProps }: AppProps) {
  return (
    // Wrap your application with the WagmiConfig component
    // and pass the client instance as a prop to it.
    <WagmiConfig client={client}>
      <Component {...pageProps} />
    </WagmiConfig>
  );
}
```

We now have our networks configured, our next step is to allow users to choose which wallet to connect to. As you can see above, we have 4 wallet connectors set up. MetaMask, WalletConnect, Coinbase and Injected(which again is basically your default wallet).

### Step 3: Use useConnect to enable picking the browser wallet

On **pages/index.tsx** copy and paste the following code. Make sure you add the CSS, so it looks good!

```tsx
import { useAccount, useConnect } from "wagmi";
import { useEffect } from "react";
import styles from "@/styles/Home.module.css";

export default function Home() {
  const { connect, connectors } = useConnect();
  const { isConnected } = useAccount();

  useEffect(() => {
    console.log(
      `Current connection status: ${isConnected ? "connected" : "disconnected"}`
    );
  }, [isConnected]);

  return (
    <>
      <p
        className={styles.status}
        style={{
          color: isConnected ? "green" : "red",
        }}
      >
        {" "}
        {isConnected !== undefined
          ? isConnected
            ? "Connected"
            : "Disconnected"
          : "loading..."}
      </p>
      <div className={styles.maincontainer}>
        <div className={styles.container}>
          <div className={styles.buttonswrapper}>
            <div className={styles.buttonswrapperGrid}>
              {connectors.map((connector) => (
                <button
                  suppressHydrationWarning
                  key={connector.id}
                  onClick={() => connect({ connector })}
                  className={styles.button28}
                >
                  {connector.name}
                </button>
              ))}
            </div>
          </div>
        </div>

        {/* send funds */}

        {/* mint nft */}
      </div>
    </>
  );
}
```

### Step 4: Add CSS to make it look nice

Delete everything on **styles/globals.css** and **styles/Home.module.css**.

Copy paste the CSS code below on **styles/globals.css.**

```css
body {
    height: 100vh;
    background: rgb(11,3,48); /* For browsers that do not support gradients */
    background: linear-gradient(to bottom right,#0b0330, #5904a4);
    font-family: 'Inter Medium', sans-serif;
}
```

Copy paste the CSS code below into **styles/Home.module.css.**

```css=
.status {
  text-align: left;
  margin: 0px;
  font-family: "Inter Medium", sans-serif;
}

.maincontainer {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  height: 60%;
  width: 60%;

}

.buttonswrapperGrid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  grid-template-rows: repeat(2, 1fr);
  grid-gap: 1.5rem;
  justify-items: center;
  align-items: center;
  justify-content: center;
  margin-bottom: 40px;
  padding: 20px;
}

/* CSS */
.button28 {
  align-items: center;
  background-color: #da5fff;
  border: 2px solid #1d0321;
  border-radius: 8px;
  box-sizing: border-box;
  color: #111;
  cursor: pointer;
  display: flex;
  font-family: Inter, sans-serif;
  font-size: 16px;
  height: 55px;
  justify-content: center;
  line-height: 24px;
  max-width: 75%;
  padding: 0 25px;
  position: relative;
  text-align: center;
  text-decoration: none;
  user-select: none;
  -webkit-user-select: none;
  touch-action: manipulation;
  width: 100%;
}

.button28:after {
  background-color: #210c20;
  border-radius: 8px;
  content: "";
  display: block;
  height: 48px;
  left: 0;
  width: 100%;
  position: absolute;
  top: -2px;
  transform: translate(8px, 14px);
  transition: transform 0.2s ease-out;
  z-index: -1;
}

.button28:hover:after {
  transform: translate(0, 0);
}

.button28:active {
  background-color: #ffdeda;
  outline: 0;
}

.button28:hover {
  outline: 0;
}

@media (min-width: 768px) {
  .button28 {
    padding: 0 40px;
  }
}

.myinput {
  background-color: rgb(253, 232, 255);
  border-radius: 15px;
  border: 2px solid #1d0321;
  padding: 20px;
  width: 44%;
  height: 15px;
}

.inputcontainer {
  display: flex;
  justify-content: space-between; /* Space the input fields evenly */
  align-items: center; /* Align the input fields vertically in the container */
}

.buttoncontainer {
  display: flex;
  justify-content: center; /* Center the button horizontally */
  align-items: center; /* Center the button vertically */
  margin-top: 16px;
  margin-bottom: 16px;
}
.mintcontainer {
  display: flex;
  justify-content: center; /* Center the button horizontally */
  align-items: center; /* Center the button vertically */
  margin-top: 50px;
  margin-bottom: 16px;
}
/* CSS */
.button64 {
  align-items: center;
  background-image: linear-gradient(144deg, #af40ff, #280b36 50%, #e30eff);
  border: 0;
  border-radius: 8px;
  box-shadow: rgba(151, 65, 252, 0.2) 0 15px 30px -5px;
  box-sizing: border-box;
  color: #ffffff;
  display: flex;
  font-family: Phantomsans, sans-serif;
  font-size: 20px;
  justify-content: center;
  line-height: 2em;
  max-width: 100%;
  min-width: 140px;
  padding: 3px;
  text-decoration: none;
  user-select: none;
  -webkit-user-select: none;
  touch-action: manipulation;
  white-space: nowrap;
  cursor: pointer;
}

.button64:active,
.button64:hover {
  outline: 0;
}

.button64 span {
  background-color: rgb(5, 6, 45);
  padding: 16px 24px;
  border-radius: 6px;
  width: 100%;
  height: 100%;
  transition: 300ms;
}

.button64:hover span {
  background: none;
}

@media (min-width: 768px) {
  .button64 {
    font-size: 24px;
    min-width: 196px;
  }
}

/* CSS */
.button49,
.button49:after {
  width: 150px;
  height: 76px;
  line-height: 78px;
  font-size: 20px;
  font-family: 'Bebas Neue', sans-serif;
  background: linear-gradient(45deg, transparent 5%, #ff01ee 5%);
  border: 0;
  color: #fff;
  letter-spacing: 3px;
  box-shadow: 6px 0px 0px #00E6F6;
  outline: transparent;
  position: relative;
  user-select: none;
  -webkit-user-select: none;
  touch-action: manipulation;
}

.button49:after {
  --slice-0: inset(50% 50% 50% 50%);
  --slice-1: inset(80% -6px 0 0);
  --slice-2: inset(50% -6px 30% 0);
  --slice-3: inset(10% -6px 85% 0);
  --slice-4: inset(40% -6px 43% 0);
  --slice-5: inset(80% -6px 5% 0);

  content: 'GET YOUR NFT';
  display: block;
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: linear-gradient(45deg, transparent 3%, #00E6F6 3%, #00E6F6 5%, #c401ff 5%);
  text-shadow: -3px -3px 0px #F8F005, 3px 3px 0px #00E6F6;
  clip-path: var(--slice-0);
}

.button49:hover:after {
  animation: 1s glitch;
  animation-timing-function: steps(2, end);
}

@keyframes glitch {
  0% {
    clip-path: var(--slice-1);
    transform: translate(-20px, -10px);
  }
  10% {
    clip-path: var(--slice-3);
    transform: translate(10px, 10px);
  }
  20% {
    clip-path: var(--slice-1);
    transform: translate(-10px, 10px);
  }
  30% {
    clip-path: var(--slice-3);
    transform: translate(0px, 5px);
  }
  40% {
    clip-path: var(--slice-2);
    transform: translate(-5px, 0px);
  }
  50% {
    clip-path: var(--slice-3);
    transform: translate(5px, 0px);
  }
  60% {
    clip-path: var(--slice-4);
    transform: translate(5px, 10px);
  }
  70% {
    clip-path: var(--slice-2);
    transform: translate(-10px, 10px);
  }
  80% {
    clip-path: var(--slice-5);
    transform: translate(20px, -10px);
  }
  90% {
    clip-path: var(--slice-1);
    transform: translate(-10px, 0px);
  }
  100% {
    clip-path: var(--slice-1);
    transform: translate(0);
  }
}

@media (min-width: 768px) {
  .button49,
  .button49:after {
    width: 200px;
    height: 86px;
    line-height: 88px;
  }
}
```

### Step 5: Run the website and test it

When you run this, this should initiate a connection to your wallet. Once you’ve approved the connection, it should look something like this:

```bash
npm run dev
```

![website preview](https://static.wixstatic.com/media/935a00_a3db98b80fa14491b0998b8e12847408~mv2.png/v1/fill/w_592,h_151,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_a3db98b80fa14491b0998b8e12847408~mv2.png)

### Step 6: Add ability to transfer cryptocurrency

Now that we’ve connected our wallet, we can transfer funds from our wallet to others. Make sure you have some Mumbai MATIC in your wallet. You can get some here [https://faucet.polygon.technology](https://faucet.polygon.technology/)

We’ll now be creating the input fields and send button for the transaction. Create a new file under **pages**/ directory and name it **RareSend.tsx**. Copy paste the code below. The explanations are in the code comments.

```tsx
import { parseEther } from "ethers/lib/utils.js";
import React, { useState } from "react";
import { useDebounce } from "use-debounce";
import { usePrepareSendTransaction, useSendTransaction } from "wagmi";
import styles from "@/styles/Home.module.css";

// what this does is simply disable the SendFunds function if the value passed is false
interface SendFundsProps {
  disabled?: boolean;
}

export default function SendFunds(props: SendFundsProps) {
  // declare two state variables for the recipient and the amount
  const [to, setTo] = useState("");
  const [debouncedTo] = useDebounce(to, 500); // useDebounce() hook to debounce the recipient input

  const [amount, setAmount] = useState("");
  const [debouncedAmount] = useDebounce(amount, 500); // useDebounce() hook to debounce the amount input

  // use the usePrepareSendTransaction() hook to prepare a transaction request
  const { config } = usePrepareSendTransaction({
    request: {
      to: debouncedTo, // recipient from debounced input
      value: debouncedAmount ? parseEther(debouncedAmount) : undefined, // amount from debounced input
    },
  });

  // use the useSendTransaction() hook to create a transaction and send it
  const { sendTransaction } = useSendTransaction(config);

  return (
    <>
      <form
        onSubmit={(e) => {
          e.preventDefault();
          sendTransaction?.(); // if sendTransaction is defined, execute it
        }}
      >
        <div className={styles.inputcontainer}>
          <input
            aria-label="Recipient"
            onChange={(e) => setTo(e.target.value)} // update the recipient state on input change
            placeholder="Address destination"
            value={to}
            className={styles.myinput}
          />
          <input
            aria-label="Amount (ether)"
            onChange={(e) => setAmount(e.target.value)} // update the amount state on input change
            placeholder="enter amount"
            value={amount}
            className={styles.myinput}
          />
        </div>
        <div className={styles.buttoncontainer}>
          <button
            disabled={!sendTransaction || !to || !amount}
            className={styles.button64}
          >
            Send
          </button>{" "}
          {/* disable the button if required fields are empty */}
        </div>
      </form>
    </>
  );
}
```

**useDebounce() explained**

To avoid overloading the RPC and getting rate-limited, we’ll limit the use of **usePrepareContractWrite** hook, which requests gas estimates on component mount and args modification. We use **useDebounce** hook in the component to delay updating the token ID by 500ms if no changes have been made.

### Step 7: Insert sendFunds component

Next, simply add the <SendFunds disabled={!isConnected}/> like this in **pages/index.tsx**.

```tsx
import { useAccount, useConnect } from "wagmi";
import SendFunds from "./RareSend";
import { useEffect } from "react";
import styles from "@/styles/Home.module.css";

export default function Home() {
  {
    /* some code... */
  }

  return (
    <>
      <div>
        {/* some code... */}

        {/* send funds */}
        <SendFunds disabled={!isConnected} />

        {/* mint nft */}
      </div>
    </>
  );
}
```

As long as you have **Matic** in your address, you’re able send it to another account!

This should be how your website looks like:

![react js send cryptocurrency](https://static.wixstatic.com/media/935a00_09ac4c32a4da47e986e936ac012b16b2~mv2.png/v1/fill/w_592,h_226,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_09ac4c32a4da47e986e936ac012b16b2~mv2.png)

Congratulations on reaching this point! You have successfully created your own website that can send funds from your wallet to another account. You are now able to easily transfer cryptocurrency between accounts, without having to rely on a third-party service or manually entering transaction details. Well done! It’s that easy!

## Wagmi Examples Part 2: Minting an NFT with React + Wagmi

We assume you already have deployed an NFT [smart contract](https://www.rareskills.io/post/solana-smart-contract-language) to the blockchain. You can follow this video tutorial to do so: [https://youtu.be/LIoFbudNVZs](https://youtu.be/LIoFbudNVZs).

![NFTs of AI generated cats](https://static.wixstatic.com/media/935a00_42ab1084a2d541b680e973113ba8c88b~mv2.png/v1/fill/w_592,h_247,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_42ab1084a2d541b680e973113ba8c88b~mv2.png)

NFTs of AI generated cats

The mint function can be created as follows. Create a **mint.tsx** and add the codes below.

```tsx
// Initialize ethers.js and wagmi dependencies
import { ethers } from "ethers";
import * as React from "react";
import {
  usePrepareContractWrite,
  useContractWrite,
  useWaitForTransaction,
} from "wagmi";
import styles from "@/styles/Home.module.css";

// Define a React function component to mint an NFT
export function MintNFT() {
  // Prepare the contract write configuration by providing the contract's address, ABI, function name, and overrides
  const { config } = usePrepareContractWrite({
    address: "Your Contract Address",
    abi: [
      {
        name: "mint",
        type: "function",
        stateMutability: "payable",
        inputs: [],
        outputs: [],
      },
    ],
    functionName: "mint",
    overrides: {
      from: "Your Wallet Address",
      value: ethers.utils.parseEther("0.000000001"), //the integer value should match your nft minting requirements
    },
  });

  // Use the useContractWrite hook to write to the contract's mint function and obtain the transaction data and write function
  const { data, write } = useContractWrite(config);

  // Use the useWaitForTransaction hook to wait for the transaction to be mined and return loading and success states
  const { isLoading, isSuccess } = useWaitForTransaction({
    hash: data?.hash,
  });

  // Render the component with a button that triggers the mint transaction when clicked, a loading message while the transaction is in progress, and a success message when the transaction is successful
  return (
    <div className={styles.mintcontainer}>
      <button
        disabled={!write || isLoading}
        onClick={() => write?.()}
        className={styles.button49}
      >
        {isLoading ? "Minting..." : "Mint"}
      </button>
      {isSuccess && (
        <div>
          Successfully minted your NFT!
          <div>
            <a href={`https://etherscan.io/tx/${data?.hash}`}>Etherscan</a>
          </div>
        </div>
      )}
    </div>
  );
}
```

Now you can simply insert it into the **index.tsx** file like this:

```tsx
import { useAccount, useConnect } from "wagmi";
import SendFunds from "./RareSend";
import { useEffect } from "react";
import styles from "@/styles/Home.module.css";
import { MintNFT } from "./mint";

export default function Home() {
  const { connect, connectors } = useConnect();
  const { isConnected } = useAccount();

  useEffect(() => {
    console.log(
      `Current connection status: ${isConnected ? "connected" : "disconnected"}`
    );
  }, [isConnected]);

  return (
    <>
      <p
        className={styles.status}
        style={{
          color: isConnected ? "green" : "red",
        }}
      >
        {" "}
        {isConnected !== undefined
          ? isConnected
            ? "Connected"
            : "Disconnected"
          : "loading..."}
      </p>
      <div className={styles.maincontainer}>
        <div className={styles.container}>
          <div className={styles.buttonswrapper}>
            <div className={styles.buttonswrapperGrid}>
              {connectors.map((connector) => (
                <button
                  suppressHydrationWarning
                  key={connector.id}
                  onClick={() => connect({ connector })}
                  className={styles.button28}
                >
                  {connector.name}
                </button>
              ))}
            </div>
          </div>
        </div>

        {/* send funds */}
        <SendFunds disabled={!isConnected} />

        {/* mint nft */}
        <MintNFT />
      </div>
    </>
  );
}
```

Your final website should look something like this:

![wagmi react js mint nft transfer crypto](https://static.wixstatic.com/media/935a00_e589bf40a5c04c89a45bc5ac7d1d3da1~mv2.png/v1/fill/w_592,h_284,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_e589bf40a5c04c89a45bc5ac7d1d3da1~mv2.png)

Congratulations! You’ve just made your own Decentralized Application that is capable of Sending Transactions and Minting an NFT.

*Originally Published April 24, 2023*

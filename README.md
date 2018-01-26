# Kyte XRP Wallet

Awesome XRP Wallet; VueJS, NodeJS / Webpack / SCSS / ripple-lib, etc.

Work in progress ;)

This is a **BETA RELEASE** to test the tech. UI/UX will be fixed, samples [here (screenshot)](https://n47esjf.dlvr.cloud/image.png) and [here (animation)](https://krga9ir.dlvr.cloud/testy%20(2).mp4).

Code is stable and OK for now. Code will be cleaned up in the near future. However: app is 100% functioning, you can:

- Add (read only) wallets by public key (wallet address)
- Add wallets by:
    - Secret key (sXXXXXX...)
    - HEX secret key
    - Mnemonic words (ledger, etc.)
- List last 100 transactions
- Create **and send** transactions

## Security

If you add a wallet using your secret or memonic, you will be asked to enter a passphrase. Your passphrase should contain at least 6 words. 

Your secret key will be saved on your local device, but **it will be encrypted using your passphrase. So please pick a safe one**.
You can always check your transactions and balance, however: if you want to submit a transaction you'll have to enter your passphrase.

Want to know how we encrypt your secret? Check the `encrypt` method at `/src/components/AddWallet.vue`. We use `CryptoJS.AES` for this.

## Most recent build

The most recent build will be published at **[https://kyteapp.co](https://kyteapp.co)** (static HTML+CSS+JS at DigitalOcean). Please check the network-console: after the inital download of HTML+CSS+JS no connections to the internet will be made except for the connections to _rippled_ (`s1.ripple.com` and `s2.ripple.com`)

**PLEASE NOTE!!** The app is in BETA, the design and code is not ready yet!

### Todo (not ready, still working on it!)

- Code Cleanup
- Air Gapped TX generation
- Air Gapped TX (in), online submit
- iOS / Android App using PhoneGap
- Desktop (Windows / Linux / OSX) app using Electron

### Known limitations (BETA)

- **Design not as [promised](https://n47esjf.dlvr.cloud/image.png)** (we started with the tech., design live in a few weeks)
- Only USD and EUR supported (todo: add multiple currencies)
- Exchange rate only from Bitstamp (todo: settings with multiple exchange rate sources)
- Limited offline functionality
- No native app, not in app stores

## Why would you trust this app?

That's up to you, however: the app:

- Is open source
- Doesn't connect to the internet, besides the connection to rippled-servers (ledger)
- Is created by XRP enthusiasts (created the [XRP Tip Bot for Reddit](https://www.xrptipbot.com) as well

## What _NOT_ to expect

In-app **wallet generation**. Kyte will never generate your wallet. Please generate your wallet somewhere else (offline) and keep your ***secret safe***!!!

## Build Setup

If you don't trust the public build, checkout & build your own:

``` bash
# install dependencies
npm install

# serve with hot reload at localhost:8080
npm run dev

# build for production with minification
npm run build
```

Check the `/dist/` folder after building ;)
# Signing In & Transactions

Using the app means signing a few different things along the way. Two are free wallet signatures, one is a signature the server makes on your behalf, and the rest are paid on-chain transactions.

### Free signatures (no gas)

- **Sign in**: right after you connect your wallet, and again if you switch accounts or after your session expires (12 hours). This lets the app recognize your account, which is required before you can write or open [notes](notes.md).
- **Unlock notes**: the first time you read or write a note in a tab. This derives the key the app uses to encrypt and decrypt your notes; it stays in that tab's memory only, so you'll sign again in a new tab.

Both just prove you control your wallet. Neither spends gas or grants any spending permission.

### A signature made for you

When an invoice is paid, the app asks the platform's server for a one-time fee address, signed by the platform. You never see this signature; the payment simply won't go through without a valid one.

### On-chain transactions (cost gas)

- Creating, paying, accepting or rejecting, canceling, releasing, and refunding invoices (paying with a token also needs an approval transaction first).
- Publishing your notes key on-chain, once.
- Admin actions: multisig propose, approve, execute, and cancel; the emergency pause and unpause.

Writing a note or marking one opened is the exception: the app sends that transaction and covers the gas for you.

Expect a wallet popup for each transaction above; the app shows an on-screen hint when one is expected.

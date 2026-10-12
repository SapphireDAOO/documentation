# Notes

Notes are short messages attached to an invoice to add context for you or the other party. They are optional, do not change the invoice amount or status, and are meant to keep payment conversations clear and organized.

For privacy, each note is locked (encrypted) before it is stored. The app generates a fresh random “scramble code” for every note and uses your secret key to encrypt the message, so what gets saved looks like unreadable text. When you view the invoice with the same secret key, the app uses that scramble code to unlock (decrypt) the note back into the original message.

Your secret key isn't stored anywhere: the first time you read or write a note, your wallet asks you to sign a free message (no gas) that the app turns into that key. You only sign this once per tab; the key then stays in memory for the rest of the session. Writing or opening a note also requires being signed in (see [Signing In & Transactions](signing-in-and-transactions.md)).

### Where you can add notes

* Create invoice: add a note when requesting payment.
* Pay page: add a note when completing a payment.
* Checkout (intermediated platform): add a note before paying with a token.
* Invoice details: use the Notes section to add follow‑up messages.

### Sharing options

When you write a note, choose whether to share it:

* Shared note: visible to both the creator and the payer.
* Private note: only visible to you in your dashboard.

If the creator shares their first note, it appears on the payment page so the payer sees it before completing payment.

### How notes appear

Notes show in the Notes section on each invoice. Shared notes can be opened and read by the other party, while private notes remain personal.

### Best practices

* Keep notes short and specific (e.g., references).
* Avoid sensitive information (secrets, passwords, or private data).
* Use notes to clarify context, not to negotiate terms.

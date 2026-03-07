# Use The Advanced Processor

### For Sellers

The marketplace creates invoices on the seller's behalf and provides details such as the seller's address and the invoice price to the payment processor. The marketplace generates a single invoice or meta-invoice with a unique ID and a link or QR code, which you share with the Buyer.

You can check the invoice status on your dashboard in SapphireDao. Invoices can be:

* **Awaiting payment** – Awaiting payment from the buyer
* **Paid** – Invoice has been paid by the buyer.
* **Expired** – The invoice expired because it was not paid within the configured time window.
* **Released** – Funds have been sent to the seller
* **Disputed** - Buyer has raised a dispute after payment.
* **Dispute Resolved** - Dispute has been resolved in full favor of both parties.
* **Dispute Dismissed** - Dispute has been dismissed without changes to payouts.
* **Dispute Settled** - Dispute has been settled with a split payout.
* **Canceled** – Invoice was canceled before payment.
* **Refunded** – Funds have been refunded to the buyer (for example, after expiration or rejection).

After the buyer makes payment, the invoice moves to Paid and the funds are held in an escrow contract. At this point, the invoice enters the release flow, since the contract automatically sets a release time and starts the release countdown. Before the funds are released, a dispute can be raised, which moves the invoice to Disputed and pauses the release process until the dispute is handled.

Refunds are handled from escrow while the invoice is in the Paid state. Depending on the situation, the authorized operator can refund part or all of the escrowed funds back to the buyer.

Once the release time is reached, payment is automatically released, and the contract pays the seller from escrow minus the platform fee. The invoice status then updates to Released.

If the buyer does not pay before the invoice expiration time, the invoice can no longer be paid. In this contract, only the authorized third-party operator via the platform operator wallet can create invoices, trigger releases, process refunds, and manage disputes.

### For Buyers

A third-party commerce platform directs the buyer to the SapphireDao payment page using the invoice ID. The page displays the invoice details, including the amount (priced in USD and shown as an equivalent crypto amount), seller information, the expiration date, and the current status (for example, Created).

The buyer pays using a crypto wallet and can pay with the native token (ETH) or another supported token. For a single invoice, the buyer pays the exact amount shown. For a meta-invoice, the buyer pays the total amount that covers all sub-invoices. Once the transaction is confirmed, funds are held in escrow and the invoice status updates to Paid.

If a refund is needed before release, it is processed from escrow through the platform’s flow. Refunds may apply if the invoice is canceled, expires, or if the third-party initiates a refund while the invoice is still in a refundable state. Disputes can be raised after payment and are handled through the dispute process, which is covered separately.

### How the Process Works

A authorized third-party commerce platform creates a single invoice or a meta-invoice, assigns a unique ID, and shares a payment link with the buyer. The buyer pays through the platform using the native token (ETH) or another supported token. Once the transaction is confirmed, the funds are held in escrow and the invoice state updates to **Paid**.

After payment, the invoice follows the release flow. A release time is set automatically, and the SapphireDAO can adjust the release time before funds are released. If a refund is required while the invoice is still in the Paid state, it is processed from escrow back to the buyer through the authorized operator. When the release time is reached, the contract automatically processes the release (via [Chainlink Automation](https://docs.chain.link/chainlink-automation)) and pays the seller from escrow minus the platform fee, updating the invoice status to **Released**.

### Important Notes

Funds are secured in escrow until released or refunded. Prices are converted from US dollars to crypto using trusted price feeds([chainlink data feeds](https://docs.chain.link/data-feeds/price-feeds/)). Only the authorized third-party platform can create invoices and manage escrow actions such as refunds, dispute handling, and releases, and it performs these actions through its designated [Platform Operator](../user-docs/publish-your-docs.md#platform-marketplace-operator-wallet). Seller payouts are made from escrow minus a small platform fee. Unpaid invoices expire after a configured validity period visible on the dashboard. Disputes, covered elsewhere, can be raised after payment and handled through the contract’s dispute flow.

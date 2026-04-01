# Use The Simple Processor

### For Sellers

Connect your wallet to SapphireDao. Navigate to **Create Invoice** to create an invoice. Enter the payment amount in ETH, optionally add a [note](notes.md), and submit the Invoice. The system generates a payment link, and a QR code to share with the Buyer.

Track invoices on the dashboard’s Invoice List, where users can monitor status like

* **Awaiting payment** – Awaiting payment from the buyer.
* **Expired** – The invoice expired because it was not paid within the configured time window.
* **Paid** – The Buyer has sent the payment.
* **Accepted** – Seller has accepted the payment.
* **Rejected** – Seller explicitly rejected the payment; buyer receives a refund.
* **Refunded** – Funds were returned from escrow to the buyer (for example, after rejection or no seller action within the allowed time).
* **Released** – Funds have been released to the seller.
* **Locked** – All automated withdrawal attempts failed; funds are secured and can be recovered by the platform admin.
* **Canceled** – The invoice was canceled.

When the Buyer makes payment, the funds are held in escrow. After reviewing the payment, the seller decides whether to accept it and moves the invoice to the release period, rejecting it to refund the Buyer. The seller has a limited time window to take action;  if not action is taken within that period, the escrow funds are automatically refunded to the Buyer.

Once the payment is accepted, a default release time is set, which can be adjusted by the platform Admin. After this period, the funds are released automatically to the seller, and the invoice status updates to Released once the release is triggered.

{% hint style="info" %}
**Key  Considerations for Creators**

* The platform Admin may adjust the release period before funds are released.
* Invoices expire if not paid within a limited time.
{% endhint %}

### For Buyers

The seller provides a QR code or payment link, which displays the invoice details, including the amount in ETH and any notes shared by the seller. The buyer pays the invoice in ETH using a cryptocurrency wallet, and also has the option to add a note (for example, a payment reference or message). Once the transaction is confirmed, the invoice status updates to **Paid**.

{% hint style="info" %}
Key Consideration for Buyers

* If the Seller cancels the invoice or it expires, invoice can no longer be paid.
* Payments are secure in escrow until the Seller approves and the release period ends.
{% endhint %}

### How the Process Works

SapphireDao generates a unique URL and QR code, which the Seller shares with the Buyer after the Seller creates an invoice with the required information. The invoice is updated to Paid when the Buyer makes payment using the link or QR code, payment to escrow. After reviewing the payment, the Seller has a limited time to decide whether to accept or reject it; if no action is taken, the money is returned. After acceptance, the platform administrator can change the default release period. Funds are moved to the Seller's wallet after the period, and the invoice is marked as released.

### Important Notes

The platform Admin can adjust the release period before funds are released. Funds are held securely in an escrow wallet until conditions are met. Seller action windows and other time limits are enforced by the smart contract and are displayed on the dashboard.

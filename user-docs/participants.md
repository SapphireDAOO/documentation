# Participants

### Seller

A seller is a participant who provides goods or services and receives payment through the system. In the simple processor, after the buyer has **made payment**, the seller accepts or rejects the payment. In the intermediated processor, payment-flow actions on the seller's behalf depend on the integrating third-party platform.

* In the simple processor, the seller creates the invoice
* in the intermediated processor, the system assigns it to the seller.

### Buyer

A buyer is a participant who pays for goods or services by interacting with an invoice, either single or part of a meta-invoice. The buyer initiates payment, and is eligible for refunds or to raise disputes if issues arise.

### Intermediated Platform Operator Wallet

The Intermediated Payment Processor contract designates an authorized platform operator wallet address. This wallet oversees invoice creation and resolution workflows. Only the platform operator wallet can perform sensitive actions such as releasing payments, resolving disputes, and creating single or meta-invoices.

<div align="center" data-full-width="false"><figure><img src="../.gitbook/assets/image (1).png" alt="A sequence diagram showing the interaction between four entities: Intermediated Platform, API, Intermediated Platform Wallet, and Smart Contract. The flow begins with the Intermediated Platform initiating a request (e.g., create, release, dispute, cancel, refund), which is passed to the API. The API instructs the Intermediated Platform Wallet to perform the action, which then sends a transaction to the Smart Contract. Once the Smart Contract confirms execution, the result is returned to the Intermediated Platform."><figcaption></figcaption></figure></div>

The platform operator wallet is configured during deployment or by the administrators, its address is granted exclusive access to these privileged functions via internal permission checks. This setup ensures that only the authorized third-party backend can initiate critical actions.

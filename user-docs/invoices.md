# Invoices

An invoice contains the relevant order data and acts as the on-chain record of a specific buyer-seller order. In the Advanced Payment Processor, an invoice can also be a meta-invoice, which combines multiple sub-invoices into a single payment.

### Meta-Invoice

This feature enables the buyer to pay for multiple orders from a single seller or multiple sellers in a single transaction. A meta-invoice combines invoices, known as **sub-invoices**, from one or more sellers to shows the buyer's entire order.&#x20;

#### Sub-Invoice

A sub-invoice represents a single order within a meta-invoice. Each sub-invoice is unique from others and contains information related to an order placed with a specific seller.

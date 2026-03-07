# Release & Refunds

SapphireDAO uses [Chainlink Automation](https://docs.chain.link/chainlink-automation) to automatically release eligible payments when their release time is reached. When an invoice is paid, the contract assigns it a timestamp and inserts the invoice into an on-chain priority queue (min-heap) ordered by the earliest release time.

Chainlink Automation continuously checks the contract to see whether the next scheduled invoice has reached its release time. When it has, Automation triggers the contract to process the queue, release the escrowed funds to the seller (minus the platform fee), and update the invoice status to Released. After the invoice is processed, it is removed from the schedule so it cannot be processed again.

Refunds are handled separately through the platform’s authorized flows, while Chainlink Automation ensures timed releases happen reliably and without manual intervention.

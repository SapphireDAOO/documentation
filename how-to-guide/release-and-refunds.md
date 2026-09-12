# Release & Refunds

{% hint style="warning" %}
**Simple Processor only.** The automated, keeper-driven release/refund/retry/burn flow described below applies to the Simple Processor. The Intermediated Processor has no automated retry or burn path: release and refund there are always triggered manually by the authorized platform operator (see [Use The Intermediated Processor](use-the-intermediated-processor.md)).
{% endhint %}

SapphireDAO automatically releases eligible payments when their release time is reached, driven by a keeper network (Chainlink CRE or Gelato Web3 Functions) through a dedicated automation adapter (replacing the earlier direct Chainlink Automation setup). The escrow hold period is fixed on the [Simple Processor](../technical-docs/core-contracts/simplepaymentprocessor.sol.md#escrow_hold_period) when it is deployed and is the same for every invoice it creates; the invoice's release time is `acceptance time + that hold period`, and the invoice is inserted into an on-chain [priority queue (min-heap)](../technical-docs/library/taskqueuelib.sol.md) ordered by the earliest release time. Nobody can change that period afterwards, including the platform owner; changing it means redeploying the processor.

The keeper periodically checks whether the next scheduled invoice has reached its release time and, when it has, triggers the [automation adapter](../technical-docs/core-contracts/paymentautomation.sol.md) to process the queue. The adapter itself holds no invoice state or funds; it just relays the trigger into the payment processor, which releases the escrowed funds to the seller (minus the platform fee) and updates the invoice status to Released. After the invoice is processed, it is removed from the schedule so it cannot be processed again. Only one keeper network is meant to be active at a time; the other exists purely as redundancy in case the primary stalls. As a further fallback if neither keeper is running, the platform owner can trigger the same processing manually.

If an automated release transfer fails, the contract retries it up to three times. If all seller-payout retries fail, the contract attempts to refund the buyer instead. If those attempts also fail, the invoice transitions to **Burned**: the escrowed funds are sent to `address(0)` and permanently destroyed.

The Administrators owner can also pause the system entirely (or a designated emergency pauser can trigger a temporary, self-expiring pause); while paused, no payment, release, refund, dispute, or cancellation action can move funds on either processor.

# Manage Admin Transactions

Privileged actions on the payment processors, such as pausing the system, updating fee settings, or changing which addresses hold admin roles, are not controlled by a single account. They go through a multisig: a defined group of admins where a transaction only takes effect once enough of them agree. Day to day, an admin works through the multisig section of the dashboard rather than interacting with the contract directly.

### Viewing and proposing

An admin opens the multisig section to view the current list of transactions (Proposed, Approved, Executed, Canceled) alongside their approval counts and proposers. To initiate a new administrative action, the admin opens the propose form, selects the target contract and function, fills in the parameters, and submits. This records the transaction as Proposed, with the proposer's own approval counted immediately.

### Notification

Every admin action, proposing, approving, executing, or canceling, triggers a notification to a configured Discord channel containing the transaction details: the transaction hash, the target contract, the decoded function call, who acted, and the current approval count. This keeps every admin aware of pending and completed actions without needing to check the dashboard.

### Approval or cancellation

Once notified, other admins review the proposed transaction, its target, decoded call, and current approval count, and decide whether to approve it or propose its cancellation:

* To approve, an admin approves from the transaction detail view. The approval is recorded and broadcast through the same Discord notification flow.
* To reject the change, an admin instead proposes a cancellation of the transaction. Cancellation is only ever carried out by the multisig itself, not by an individual admin acting alone, so it goes through this same propose, approve, execute flow rather than being a direct action any one admin can take.

### Execution

Once the number of approvals reaches the configured threshold, usually the majority of admins, the transaction becomes eligible to run and any admin can execute it. Executing performs the underlying call on the target contract and marks the transaction Executed. A transaction can never be executed before its threshold is met, and never executed twice.

### Example: adding a new admin

1. **Propose.** To propose adding a new admin, an admin goes to the multisig section, opens the propose form, targets the multisig itself, chooses the add-admin action, enters the new admin's address, and submits. This only starts the process; it does not add the new admin on its own. The transaction appears as Proposed, with the proposer already counted as one approval.
2. **Notify.** The Discord channel receives a notification with the transaction details: what it does (adding the given address as an admin), who proposed it, and the current approval count.
3. **Approve or cancel.** The other admins see the notification and open the transaction to review it. Each one either approves it, adding their approval to the count, or, if they disagree with the change, proposes a cancellation of that same transaction instead.
4. **Execute.** Once approvals reach the threshold, usually a majority of admins, the transaction is Approved and any admin can execute it. Executing runs the underlying call, the new address becomes an admin, and the transaction is marked Executed. If a cancellation was proposed and reaches the threshold first, the original add-admin transaction is Canceled instead, and the new address is never added.

For the underlying contract mechanics, function signatures, and events, see [MultiSig.sol](../technical-docs/core-contracts/multisig.sol.md).

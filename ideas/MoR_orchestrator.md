### Product

> Refer to: https://github.com/juspay/hyperswitch-prism

MoR orchestrator inspired by hyperswitch/prism, providing the same convenience for MoRs insteads of payment gateways

* Build a **stateless Merchant of Record (MOR) orchestrator library**.
* The library sits between the application and MORs such as Dodo, Paddle, and future MORs.
* The application should integrate with the orchestrator rather than directly integrating with individual MORs.
* Initially support **Dodo and Paddle**, but design the system from day one so adding many other MORs later is straightforward.
* Adding or removing an MOR should require very little change to the application.

### Choosing the MOR

* The application should be able to define **which MOR to use and when**.
* It should be possible to choose a MOR based on:

  * Country/region
  * Currency
  * Product
  * One-time payment vs subscription
  * Customer
  * Tax requirements
  * MOR capabilities
  * Any other business rule
* Different countries or situations can have completely different MOR setups.
* Multiple MORs can be configured at the same time.
* There should be a clear fallback when the preferred MOR cannot handle a particular case.

### Switching from Dodo

* A company already using Dodo should be able to introduce the orchestrator without rebuilding its billing system.

* **Already-paid payments must continue to be treated as paid.**

* Existing customers must continue to work.

* Existing subscriptions should continue normally.

* Existing recurring/auto-pay arrangements should continue wherever possible.

* Existing invoices, refunds, cancellations, and payment history should remain usable.

* Existing Dodo IDs and relationships should not suddenly become invalid just because the orchestrator was introduced.

* A company should be able to start with:

  **Application → Orchestrator → Dodo**

  and later move to:

  **Application → Orchestrator → Dodo + Paddle + other MORs**

* The company should not have to rewrite its application every time it changes its MOR setup.

### Changes to MORs

* MORs will inevitably change their APIs, products, rules, and behavior.
* Those changes should be contained within that MOR's integration.
* Updating the Dodo integration should **not break existing Dodo customers or historical data**.
* Old payments and subscriptions should continue to be understood even after the Dodo integration has been updated.
* If a subscription was created using an older version of Dodo's API, the orchestrator should still know how to handle it after the integration is updated.
* New versions should be introduced without forcing existing customers to migrate immediately.
* A major change in one MOR should not affect other MOR integrations.

### Future MORs

* Adding a new MOR should primarily mean **adding its integration**, rather than redesigning the orchestrator.
* The system should be designed around the common things MORs do while allowing individual MORs to have different capabilities.
* It should be possible to support MORs that work quite differently from Dodo or Paddle.
* MOR-specific features should be supported without making the whole system dependent on that MOR.
* Removing an MOR should not destroy the application's historical records.
* If an MOR disappears or stops being supported, old transactions and subscriptions should still remain understandable.

### Important situations to handle

* Payment succeeded but the application did not receive the confirmation.
* The same event is received more than once.
* Events arrive late or in the wrong order.
* A subscription is cancelled while the MOR is being changed.
* A payment fails during a MOR change.
* A customer moves to another country.
* A customer's existing subscription cannot be moved to another MOR.
* A customer's payment method cannot be transferred to another MOR.
* Old subscriptions continue on one MOR while new subscriptions use another.
* A MOR temporarily stops working.
* The preferred MOR cannot support a particular country, currency, product, or payment type.
* A MOR changes or removes something that was previously supported.
* A new MOR has capabilities that the orchestrator did not originally anticipate.

The core idea is therefore:

> **Build one stable MOR layer for the application, while making the MOR underneath it replaceable, configurable, and future-proof—without breaking what already exists.**

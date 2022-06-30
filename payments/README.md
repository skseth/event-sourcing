# Payment Switch

## Key Functions

Payment Switch - orchestrates with Banks and PSPs, supporting two types of payments : 

- payer initiated
- payee initiated

Payments have to be

- low latency
- high availability
- high durability
- very high volumes
- adhere to business limits on risk (exposure to bank)

# Flows 

## Payer Initiated

- Payer initiates payment which goes to PSP
- PSP forwards request to Payment Switch after checking VPA

- Payment Switch first does the following :
  - Payment Switch sends debit request to payer bank
  - Swicth ensures debit / credit limits are not crossed for either bank

- Payment Switch then sends credit request to payee bank
- Payment Switch sends transaction confirmation to PSP

## Payee Initiated

- Payment Switch receives collect request from payee's PSP
- Payment switch forwards request to payer's PSP
- Rest of processing is as per payer-initiated

## Settlement

Every 5 minutes, transactions which are done upto that point are cut off and put in a settlement file

Banks settle between themselves and any pending transactions are reconciled.

Payment records have to be maintained for 5 years and should handle dispute resolution requests.

# Errors & Business Continuity

## Timeouts

There are 3 timeouts 

- If a transaction has not reached the credit leg within 2 seconds of transaction receipt, it should be canceled
- If a transaction credit leg response is not received in 2 seconds it should be canceled - the transaction is marked as 'possibly completed'
- If the PSP response to confirmation is not received in 1 second it is canceled


## Business Errors

Any failure before the credit request is considered a failure and the transaction is marked as Completed Failed.

Payments may be in the following possible states at the time of settlement :

- Completed Successfully
- Completed Failed - this is true if the credit leg has not been executed
- Possibly Completed - this is the case if the credit leg was attempted but no response was received from the bank receiving the credit

During settlement, all possibly completed transactions are reconciled and put in the completed successfully or completed failed state. The only possible issue may be when a payer bank has no debit record, and the payee bank has a credit record. In this case, the Payment Switch records will be the tie-breaker, or else the payment will be marked as a dispute and go into dispute resolution.

## Circuit Breakers

Circuit Breakers should kick in whenever certain business or technical states are reached.

If a bank debit limit is reached, all transaction with this bank as payer must be refused on receipt

If a PSP or Bank has a large number of api call failures in a short period of time, exponential backoff techniques should be followed.

The circuit breaker status of each PSP / Bank should be visible on a dashboard 

## Business Continuity

In the event of any disaster, all payment processing should switch to an alternate site with no loss of payment records in flight ot completed.

# Design

## Components

- Payment Switch - golang service 
- Settlement System - 
- Bank Limit System - golang service
- Payment Record System (maintaining database)

- Bank Payment Service (simulating banks)
- PSP Service (simulating PSPs)


# Key Design Considerations

## Low Latency w/ Business Continuity

Ideally, the flow would work as follows :

1. Request is received from PSP
2. A record of the request is written to Kafka and mirrored to remote site
3. Bank limits are checked 
4. After the mirroring + limit check, the request is sent to the payer bank
5. Then, the response to the first request is recorded, and mirrored to remote site
6. Request is made to credit bank
7. Record is made of success, and mirrored to remote site 
8. Response is sent to PSP

However, this can result in an increase in latency. A possible flow which reduces latency can increase some errors but result in higher performance, with tradeoff of occasional delay in reversing extra debits :

1. Request is received from PSP
2. Request is immediately forwarded to payer bank
3. In parallel a write is done to Kafka, with mirroring
4. Also in parallel, the bank limit is checked
5. Also in parallel, limits are checked
6. Only on success of 2 & 3, credit request is sent to credit bank
7. Response is sent to PSP
8. In parallel credit response is recorded and mirrored to remote site (without waiting for confirmation)

There are some possible inconsistencies possible, but they are acceptable from a business perspective :

- If step 2 succeeds but step 3 fails, the payment switch may have no record of the payment. But this is ok because during settlement this will be rolled back
- If step 5 succeeds but step 6 fails, the payment is in a 'possibly successful' state, but has actually succeeded. This can be reconciled during settlement

If failure rate is low, the decrease in latency is worth the marginal rise in (recoverable) business errors. No completed payment is lost, though recovery may need the settlement to be completed. In the event of errors, in upto 10 minutes, the payer / payee will know the status in the worst case, even with disaster recovery. This customer impact can be reduced further, if an additional API is available at the bank to request status of a transaction by its unique (PSP generated) id.

This leads to an eventual consistency model, where all transactions are "settled" in about 10 minutes in the worst case, but within 5 seconds in the vast majority of cases.

## Separation of Low and High Value Transactions

A key challenge is the maintenance of bank limits, which can become a central point of failure / contention if not tackled properly.

Instead, we separate payment flows by value. Rather than checking for bank limits on every transaction, we can follow a risk-based approach as follows : For low-value transactions, we work off logged transactions in the background and only refuse when the bank "circuit breaker" trips. This may lead to some excess in bank limit, but will have limited impact due to the low value.

For high value transactions, a synchronous check can be applied to ensure limits are checked. This type of approach can reduce latency at the cost of some risk of excess. In fact, a mix where the strategy shifts to fully synchronous once a threshold close to the actual limit is reached may actually allow almost complete elimination of this risk.

A similar approach can be used also for the rest of the payment flow, where low-value payments need not wait for DR-writes.

These design approaches should be managed by policy, and ideally should be adjustable on the fly with a granularity of bank + settlement cycle.

## Time stamping of incoming requests

Every five minutes, new payment requests move to a new settlement cycle. We need to ensure no clock drift exists between machines so transactions are not assigned the incorrect settlement cycle.

We can also prevent writes to the queues after time has elapsed which can result in the payment switch moving to the next cycle.

## Sharding of settlement data & dispute resolution

Payment events can be written in an "event-sourced" manner to the database. This mechanism will mean all database entries are immutable. TThis can be extended also to when dispute resolution is carried out.

Databse tables can be sharded by time and if necessary by payer bank or psp.

# TODO
https://medium.com/larus-team/how-to-setup-mirrormaker-2-0-on-apache-kafka-multi-cluster-environment-87712d7997a4
https://github.com/Vanlightly/ChaosTestingCode/blob/master/KafkaUdn/readme.md
https://www.sohamkamani.com/install-and-run-kafka-locally/
https://www.sohamkamani.com/golang/working-with-kafka/
https://stackoverflow.com/questions/29197685/how-to-close-abort-a-golang-http-client-post-prematurely
https://github.com/alitto/pond


https://martin.kleppmann.com/2012/12/05/schema-evolution-in-avro-protocol-buffers-thrift.html

https://betterprogramming.pub/an-alternative-to-outbox-pattern-7564562843ae
https://www.squer.at/en/blog/stop-overusing-the-outbox-pattern/

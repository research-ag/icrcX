# ICRC-84: Deposit and Withdrawal Standard for ICRC-1 tokens

Financial service canisters use this standard to allow users to deposit ICRC-1 tokens and withdraw them again.
An example for such a service is a DEX.

## Tokens

The same service can accept deposits in one or more different ICRC-1 tokens.
A token is uniquely identified by the principal of its ICRC-1 ledger.

```candid "Type definitions" +=
type Token = principal;
```

The list of accepted tokens can be queried with the following function.

```candid "Methods" +=
icrc84_supported_tokens : () -> (vec Token) query;
```

## Amounts

Amounts are specified as `nat` in the smallest unit of the ICRC-1 token.
Decimals do not play a role in the interface.

```candid "Type definitions" +=
type Amount = nat;
```

To get the decimals the user has to query the ICRC-1 ledger.

## Users

Users are identified by their principal.

```candid "Type definitions" +=
type User = principal;
```

## Deposit accounts

There are two ways for a user to deposit funds to the service.
The first one is via direct transfer to a deposit account.
The second one is via an allowance,
but only if the ICRC-1 ledger supports ICRC-2.

In the direct transfer method,
users make deposits into individual deposit accounts which are subaccounts that are derived from the `User` principal in a deterministic and publicly known way.
The derivation works by embedding the principal bytes right-aligned into the 32 subaccount bytes, pre-pending a length byte, and left-padding with zeros.

```candid "Type definitions" +=
type Subaccount = blob;
```

## Requirements

The only requirement on the underlying token ledger is the ICRC-1 standard.
Since the standard deposit method is based on deposit accounts, not allowances, the ICRC-2 extension is not required.
Moreover, as will become clear below,
the deposit method is _balance-based_ (as opposed to transaction-based).
This means it is sufficient that the service can read the balances in the deposit accounts from the underlying token ledger.
It is not required that the service can inspect individual deposit transactions by transaction id, memo or other means.
Hence, it is not required that the underlying token ledger provides an indexer, transaction history or archive.
In particular, the ICRC-3 extension is not required.

## TokenInfo

For each token the service has the following public set of configuration parameters defined.
The values may change over time.

```candid "Type definitions" +=
type TokenInfo = record {
  deposit_fee : Amount;
  withdrawal_fee : Amount;
  min_deposit : Amount;
  min_withdrawal : Amount;
};
```

`deposit_fee` specifies the fee that is deducted each time a deposit is detected and consolidated into the service's main account.
The `deposit_fee` can but does not have to coincide with the transfer fee of the underlying ICRC-1 token.
However, the _application_ of the `deposit_fee` should coincide with actual transfers happening.
For example, if the user makes multiple installments into the deposit account and then the service manages to consolidates them all at once into its main account then the `deposit_fee` should be charged only once.
But still, the amount of the `deposit_fee` can differ from the underlying transfer fee charged by the ledger.

`withdrawal_fee` specifies the fee that is deducted when the user makes a withdrawal. 
The `withdrawal_fee` can but does not have to coincide with the transfer fee of the underlying ICRC-1 token.
It is charged for each withdrawal that a user makes and that results in a successful ICRC-1 transfer.

`min_deposit` is the minimal deposit that is considered valid by the service.
Any balance in a deposit account that is below this value is ignored.
For example, say for the ICP token a service has defined `deposit_fee = 20_000` and `min_deposit = 100_000`.
If the user makes a deposit of exactly `100_000` e8s
then `20_000` will be deducted and the user will be credited
with `80_000` e8s. 
The service will empty out the user's deposit account.
As a result, the service will take in `90_000` e8s because the ICP ledger's transfer fee is `10_000` e8s.
If instead the user had made a deposit of `99,999` then it would have been ignored.

`min_deposit` must be larger than `deposit_fee`.
For example, if `deposit_fee = 20_000` then `min_deposit` must be at least `20_001`.

`min_withdrawal` is the minimal withdrawal that a user can make.
Any withdrawal request below this amount will be denied.
`min_withdrawal` must be larger than `withdrawal_fee`.
For example, say for the ICP token a service has defined `withdrawal_fee = 20_000` and `min_withdrawal = 100_000`.
If the user requests a withdrawal of exactly `100_000` e8s
then the user will be debited with `100_000` e8s
and the service will initiate a transfer of `80_000` e8s to the user.
As a result, the service will pay `90_000` e8s because the ICP ledger's transfer fee is `10_000` e8s.

Note: The service will never make transfers of amount 0 on the ICRC-1 ledgers even though ICRC-1 technically allows them.
This is true for consolidation of deposits and for withdrawals.

The token info can be queried with the following method.

```candid "Methods" +=
icrc84_token_info : (Token) -> (TokenInfo) query;
```

If the specified `Token` is not supported by the service then the call will throw the async error `canister_reject` with error message `"UnknownToken"`.

## Credits

Credits are tracked by the service on a per-token basis.
The unit for credits is the same as the unit of the corresponsing ICRC-1 token.
However, credits are of slighly different nature than token balances even though the use the same unit.
Credits are virtual and for greater flexibility we allow credits to go negative, hence we use type `int`.

A user can query his personal credit balance with the following method.

```candid "Methods" +=
icrc84_credit : (Token) -> (int) query;
```

If the specified `Token` is not supported by the service then the call will throw the async error `canister_reject` with error message `"UnknownToken"`.

Credit balances are private.
The above method returns the balance of the caller.

The service is not expected to distinguish non-existing users from existing ones with a credit balance of 0.
If the caller is not known to the service,
has never used the service before,
or has never used the service for the given Token before
then the method simply returns a value of zero.

For greater efficiency and to reduce query load, 
there is a method to obtain a user's credits in all tokens at once.

```candid "Methods" +=
icrc84_all_credits : () -> (vec record { Token; int }) query;
```

The returned vector contains all tokens for which the caller has a non-zero credit balance.
The tokens with a zero credit balance are stripped from the response.

As before, a non-existing user is handled the same as a user with
a zero balance in all tokens.
In both cases an empty vector is returned.

## Notification

There are two steps required when a user makes a deposit with the direct transfer method:

1. Make a transfer on the underlying ICRC-1 ledger into the personal deposit account under control of the service.
2. Notify the service about the fact that a deposit has been made.

Then the service queries the ICRC-1 ledger for the balance in the deposit account and credits the user.

The second step is done via the following method.

```candid "Methods" +=
  icrc84_notify : (NotifyArg) -> (NotifyResult);
```

where

```candid "Type definitions" +=
type NotifyArg = record {
  token : Token;
};
```


A call to `icrc84_notify` notifies the service about a deposit into the deposit account of the caller for the specified token.
The service is free to expand this record with additional optional fields to include an action that is to be done with the newly detected deposits.

The result type is as follows.

```candid "Type definitions" +=
type NotifyResult = variant {
  Ok : record {
    deposit_inc : Amount;
    credit_inc : Amount;
    credit : int;
  }; 
  Err : variant {
    CallLedgerError : record { message : text };
    NotAvailable : record { message : text };
  };
};
```

If the specified `Token` is not supported by the service then the call will throw the async error `canister_reject` with error message `"UnknownToken"`.

The service will make a downstream call to the underlying ICRC-1 ledger before returning to the user.
If the downstream call fails then the variant `Err = CallLedgerError` is returned.
The error message is not specified by this standard but is recommended to describe the async error that actually happened in the downstream call.

The service is not expected to make concurrent downstream calls for the same balance.
Hence, if the same caller calls `notify` twice concurrently for the same `Token` then the second call will return `Err = NotAvailable`.
This error generally means the `notify` method is currently blocked for this caller and token, and that it should be retried later. 
The additional text error message returned with `NotAvailable` is not specified by this standard. 

If the downstream call succeeds then the method will return the `Ok` record.

The `deposit_inc` field is the incremental deposit amount that was detected relative to the last known deposit balance.
If no new deposit was detected then a zero value is returned.

Calls to notify are not idempotent.
If the user makes one deposit transfer and then calls `notify` twice (with no additional transfer between the two calls to `notify`)
then the first call will return a non-zero `deposit_inc` value
and the second call will return zero.

If the user makes two deposit transfers and then calls `notify`
(with no additional `notify` call between the two deposit transfers)
then `notify` will return the sum of the two transfer amounts as `deposit_inc`.

The `credit_inc` field is the incremental credit amount applied to the user as a result of this call.
The value may be lower than `deposit_inc` due to the application of deposit fees, but does not have to be lower.
`credit_inc` is provided here because the user cannot reliably compute it himself from other data.

The `credit` field is the absolute credit balance after any newly detected deposit has been credited.

If multiple deposit transactions happened concurrently with calls to `notify` then the end result may depend on timing.
For example, say the ledger fee is 10 and the initial credit balance of the user is 0.
If a deposit of 20 tokens is made, then `notify` is called, then another 20 tokens are deposited and `notify` is called again
then the two `notify` responses are:
`{ deposit_inc = 20; credit_inc = 10; credit = 10 }`, 
`{ deposit_inc = 20; credit_inc = 10; credit = 20 }`.
If the first `notify` arrives _after_ the second deposit then two responses are:
`{ deposit_inc = 40; credit_inc = 30; credit = 30 }`, 
`{ deposit_inc = 0; credit_inc = 0; credit = 30 }`.
In this case the deposit fee is applied only once because the service sees it as one deposit.

The service is free to expand the response record with additional optional fields.
For example, if the service has expanded the argument record with a field specifying an action which is done after the notification
then it may want to also expand the response record with a field describing the result of that action.

## Tracked balance

It was said above that `deposit_inc` returned by `notify` is the difference in deposit balance relative to the last known (= "tracked") deposit balance.
The tracked deposit balance can be queried with the following method.

```candid "Methods" +=
icrc84_trackedDeposit : (Token) -> (BalanceResult) query;
```

If the specified `Token` is not supported by the service then the call will throw the async error `canister_reject` with error message `"UnknownToken"`.

Otherwise the method the returns the following type.

```candid "Type definitions" +=
type BalanceResult = variant {
  Ok : Amount;
  Err : variant {
    NotAvailable : record { message : text };
  };
};
```

The `Amount` returned is the currently known balance that the caller has in the specified `Token`.

For example, say a deposit flow has been interrupted during the notification step.
The user does not know if the attempted call to `notify` has gone through or not.
Then the user can query the ledger to obtain the balance in the deposit account
and can query the service to obtain the known deposit balance.
If they differ then the user must call `notify` again.

Of course, the user can call `notify` directly but the two query calls are considered cheaper and faster.
Hence this query method is provided.

If any concurrent downstream calls to the ledger are underway that could affect the returned `Amount`
then the service returns the `Err = NotAvailable` variant.
This indicates to the user to try again later.
For example, the downstream call could be a balance query (triggered by `notify`)
or a consolidation transfer that relates to the caller's deposit account for the specified `Token`.

## Deposit

An alternative way to make deposits is via allowances.
The user has to set up an allowance for one of its subaccounts with the service's principal as the spender.
The user then calls the function

```candid "Methods" +=
  icrc84_deposit : (DepositArg) -> (DepositResponse);
```

with the following argument:

```candid "Type definitions" +=
type DepositArgs = record {
  token : Token;
  amount : Amount;
  subaccount : opt Subaccount; // subaccount of the caller which has the allowance
};
```

`token` is the Token that is being deposited.
`amount` is the amount that is to be drawn from the allowance into the service.
Any ledger transfer fees will be added on the user account's side.
`subaccount` is the user's subaccount that carries the allowance where `null` means the default account.

If successful, the call returns:

* the ICRC-1 ledger txid of the transfer that happened 
* the incremental credit that resulted out of this call
* the absolute credit balance after the incremental credit has been applied 

```candid "Type definitions" +=
type DepositResponse = variant {
  Ok : DepositResult;
  Err : variant {
    AmountBelowMinimum : record {};
    CallLedgerError : record { message : text };
    TransferError : record { message : text }; // insufficient allowance or insufficient funds
  };
};

type DepositResult = record {
  txid : nat;
  credit_inc : Amount;
  credit : int;
};
```

Possible errors that can occur are:
* the amount can be lower than the minimum that the service has defined (AmountBelowMinimum)
* the ICRC-1 ledger may not support ICRC-2 (CallLedgerError)
* the inter-canister call to the ICRC-2 ledger can fail entirely (CallLedgerError)
* the call can go through but the transfer can fail (TransferError)

## Withdrawal

The user can initiate a withdrawal with the following method.

```candid "Methods" +=
icrc84_withdraw : (WithdrawArgs) -> (WithdrawResult);
```
with
```candid "Type definitions" +=
type WithdrawArgs = record {
  token : Token;
  to_subaccount : opt Subaccount;
  amount : Amount;
};
```

The `WithdrawArgs` record specifies
the `Token` to be withdrawn,
the subaccount of the caller which is the destination of the withdrawal,
and the `Amount` to be taken from the caller's credits.

If the specified `Token` is not supported by the service then the call will throw the async error `canister_reject` with error message `"UnknownToken"`.

If the specified `Subaccount` is not 32 bytes long
then the call will throw the async error `canister_reject` with error message `"InvalidSubaccount"`.

Otherwise, the following result type is returned.

```candid "Type definitions" +=
type WithdrawResult = variant {
  Ok : record {
    txid : nat;
    amount : Amount;
  };
  Err : variant {
    InsufficientCredit : record {};
    AmountBelowMinimum : record {};
    CallLedgerError : record { message : text };
  };
};
```

If the user's credit is below the requested `Amount` then `Err = InsufficientCredit` is returned.

If the requested `Amount` is smaller than the Token parameter `min_withdrawal` then `Err = AmountBelowMinimum` is returned.

If the downstream call to the ICRC-1 ledger fails with an async error then `Err = CallLedgerError` is returned.
The accompanying text message should indicate the actual async error that happened.

Otherwise the `Ok` variant is returned. 
It contains the `txid` on the underlying ICRC-1 ledger of the withdrawal transfer.
It contains the `Amount` that was actually received by the user.
In general, this `Amount` will differ from the requested amount
because `withdrawal_fee` was deducted.

## FAQ

### Why is `notify` access-controlled?

Notify is not idempotent in its return value.
If someone else can call notify for us then we could miss an incremental value.

Notify calls are expensive for the service because of the downstream inter-canister call that they trigger.
Restricting the caller makes it easier to control or charge for that cost.

### Why is the credit balance access-controlled?

Deposits are publicly visible on the ICRC-1 ledger.
Any observer can conclude from those deposit transactions 
to corresponding incoming credits for the user.
But from there on further changes to the credit balance, increase or decrease, depend on the usage of the service by the user.
For example, in a DEX the credit changes would correspond to bids placed or trades executed.
We do not want to leak that information.

### Why does `notify` use a balance-based approach, not transaction-based?

The transaction-based approach would mean that the user "claims" a specific deposit transaction where the transaction is specified by txid and is bound to the user by memo.
The advantage is that individual deposit accounts can be avoided,
hence the consolidation step is not needed which saves fees.

The disadvantages are:

* The memo field is too short to hold an entire principal, hence the service has to keep a map from user principal to an id used in the memo field.
* The service needs to store the already claimed txids forever so that they cannot be claimed a second time.

We prefer the approach that requires less state.
It makes the service leaner and easier to handle upgrades.

### Why can withdrawals only be made to the caller's account?

This is done to offload tracability of funds to the underlying ICRC-1 ledgers.
By looking at the underlying ICRC-1 ledger only,
all deposited funds can be uniquely linked to the principal that is being credited.
Through this restriction the same is possible for withdrawals.
By looking at the receiving principal of a transfer on the ledger
we know that this same principal was also the one who initiated the withdrawal. 

This prevents the service to act as a mixer.
Without this restriction,
the services could potentially be forced to either do KYC or to provide a complete log of its internal transaction,
to make the internal flow of funds tracable. 
However, we want to be able to keep the services as simple as possible.
In particular, we do not want to burden the services with the necessity to log all internal transactions.
Hence, this restrictions is made to offload all logging to the ICRC-1 ledgers.

### What are the benfits of using `notify` vs allowances?

Allowances are simpler to process for the service. 
Overall transaction fees are lower if an allowance is used for multiple deposits.

Deposits into subaccounts are provided in case:

* the ICRC-1 ledger does not support ICRC-2
* the user's wallet does not support ICRC-2 (currently most wallets)
* the user wants to make a deposit directly from an exchange

## Open questions

Shall we offer a function for a user to "burn" his credits?

Shall we offer a function for a user to retrieve history such as a log of credit/debit events?

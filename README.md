# Fungible Token Standard

This is a description of the Flow standard for fungible token contracts.  It is meant to contain the minimum requirements to implement a safe, secure, easy to understand, and easy to use fungible token contract. It also includes an example implementation to show how a concrete smart contract would actually implement the interface.

## What is Flow?

Flow is a new blockchain for open worlds. Read more about it [here](https://www.onflow.org/).

## What is Cadence?

Cadence is a new Resource-oriented programming language 
for developing smart contracts for the Flow Blockchain.
Read more about it [here](https://docs.onflow.org/docs) and see its implementation [here](https://github.com/onflow/cadence)

We recommend that anyone who is reading this should have already
completed the [Cadence Tutorials](https://docs.onflow.org/docs/getting-started-1) 
so they can build a basic understanding of the programming language.

Resource-oriented programming, and by extension Cadence, 
is the perfect programming environment for currencies, because users are able
to store their tokens directly in their accounts and transact
peer-to-peer. Please see the [blog post about resources](https://medium.com/dapperlabs/resource-oriented-programming-bee4d69c8f8e)
to understand why they are perfect for digital assets.

## Feedback

Flow and Cadence are both still in development, so this standard will still 
be going through a lot of changes as the protocol and language evolves, 
and as we receive feedback from the community about the standard.

We'd love to hear from anyone who has feedback. 
Main feedback we are looking for is:

The feedback we are looking for is:

- Are there any features that are missing from the standard?
- Are the features that we have included defined in the best way possible?
- Are there any pre and post conditions for functions that are missing?
- Are the pre and post conditions defined well enough? Error messages?
- Are there any other actions that need an event defined for them?
- Are the current event definitions clear enough and do they provide enough information for apps and other actors a clear look into what is happening?
- Are the variable, function, and parameter names descriptive enough?
- Are there any openings for bugs or vulnerabilities that we are not noticing?
- Is the documentation/comments clear and concise and organized in a coherent manner?

## Basics of the Standard:

The code for the standard is in `src/contracts/FungibleToken.cdc`. An example implementation of the standard that simulates what a simple FlowToken would be like is in `src/contracts/FlowToken.cdc`.

Example transactions that users could use to interact with fungible tokens are located in the `src/transactions/` directory.

The standard consists of a contract interface called `FungibleToken` that requires implementing contracts to define a `Vault` resource that represents the tokens that an account owns. Each account that owns tokens will have a `Vault` stored in its account storage.  Users call functions on each other's `Vault`s to send and receive tokens.  

Right now we are using unsigned 64-bit fixed point numbers `UFix64` as the type to represent token balance information. This type has 8 decimal places and cannot represent negative numbers.

## Core Features (All contained in the main FungibleToken interface)

1- Getting metadata for the token smart contract via the fields of the contract:

- `pub var totalSupply: UFix64`
    - The only required field of the contract.  It would be incremented when new tokens are minted and decremented when they are destroyed.
- Event that gets emitted when the contract is initialized
    - `pub event FungibleTokenInitialized(initialSupply: UFix64)`

2- Retrieving the token fields of a `Vault` in an account that owns tokens.

- Balance interface
    - `pub var balance: UFix64`
        - The only required field of the `Vault` type

3- Withdrawing a specific amount of tokens *amount* using the *withdraw* function of the owner's `Vault`

- Provider interface
    - `pub fun withdraw(amount: UFix64): @Vault`
        - Conditions
            - the returned Vault's balance must equal the amount withdrawn
            - The amount withdrawn must be less than or equal to the balance
            - The resulting balance must equal the initial balance - amount
    - Users can give other accounts a reference to their `Vault` cast as a `Provider` to allow them to withdraw and send tokens for them.  A contract can define any custom logic to govern the amount of tokens that can be withdrawn at a time with a `Provider`.  This can mimic the `approve`, `transferFrom` functionality of ERC20.
- withdraw event
    - Indicates how much was withdrawn and from what account the `Vault` is stored in.
      If the `Vault` is not in account storage when the event is emitted,
      `from` will be `nil`.
    - `pub event withdraw(amount: UFix64, from: Address?)`

4 - Depositing a specific amount of tokens *from* using the *deposit* function of the recipient's `Vault`

- `Receiver` interface
    - `pub fun deposit(from: @Vault)`
    - Conditions
        - `from` balance must be non-zero
        - The resulting balance must be equal to the initial balance + the balance of `from`
- deposit event
    - Indicates how much was deposited and to what account the `Vault` is stored in.
      If the `Vault` is not in account storage when the event is emitted,
      `to` will be `nil`.
    - `pub event Deposit(amount: UFix64, to: Address?)`
- Users could create custom `Receiver`s to trigger special code when transfers to them happen.

5 - Creating an empty Vault resource

- `pub fun createEmptyVault(): @Vault`
- Currently have no event
- Defined in the contract, but not in the `Vault` resource.  
  This means that to create an empty `Vault`, 
  the caller would always have to call the function in the contract.
- Conditions:
    - the balance of the returned Vault must be 0

6 - Destroying a Vault

If a `Vault` is explicitly destroyed using Cadence's `destroy` keyword, the balance of the destroyed vault must be subracted from the total supply.

7 - Standard for Token Metadata

- not sure what this should be yet
- Could be a dictionary, could be an IPFS hash, could be json, etc.
- need suggestions!


## Comparison to Similar Standards in Ethereum

This spec covers much of the same ground that a spec like ERC-20 covers, but without most of the downsides.  

- Tokens cannot be sent to accounts or contracts that don't have owners or don't understand how to use them, because an account has to have a `Vault` in its storage to receive tokens.  No `safetransfer` is needed.
- If the recipient is a contract that has a stored `Vault`, the tokens can just be deposited to that Vault without having to do a clunky `approve`, `transferFrom`
- Events are defined in the contract for withdrawing and depositing, so a recipient will always be notified that someone has sent them tokens with the deposit event.
- The `approve`, `transferFrom` pattern is not included, so double spends are not permitted
- Transfers can trigger actions because users can define custom `Receivers` to execute certain code when a token is sent.
- Cadence integer types protect against overflow and underflow, so a `SafeMath`-equivalent library is not needed.

### Metadata

A standard for token metadata is still an unsolved problem in the general blockchain world and we are still thinking about ways to solve it in Cadence. We hope to be able to store all metadata on-chain and are open to any ideas or feedback on how this could be implemented.


## Bonus Features

**Minting and Burning are not included in the standard but are included in the FlowToken example contract to illustrate what minting and burning might look like for a token in Flow.**

8 - Minting or Burning a specific amount of tokens using a specific minter resource that an owner can control

- `MintandBurn` Resource
    - function to mintTokens
    - tokens minted event
    - Each minter has a set amount of tokens that they are allowed to mint. This cannot be changed and a new minter needs to be created to add more allowance.
    - function to burnTokens
    - tokens Burnt event
    - Each time tokens are minted or burnt, that value is added or subtracted to or from the total supply.


**The following features could each be defined as a separate interface. It would be good to make standards for these, but not necessary to include in the main standard interface and are not currently defined in this example.**

9 - Withdrawing a specific amount of tokens from someone else's `Vault` by using their `provider` reference.

- approved withdraw event
- Providing a resource that only approves an account to send a specific amount per transaction or per day/month/etc.
- Returning the amount of tokens that an account can send for another account.
- Reading the balance of the account that you have permission to send tokens for
- Owner is able to increase and decrease the approval at will, or revoke it completely
    - This is much harder than anticipated

11 - Pausing Token transfers (maybe a way to prevent the contract from being imported)

12 - Cloning the token to create a new token with the same distribution

13 - Restricted ownership (For accredited investors and such)
- whitelisting
- blacklisting

# How to use the Fungible Token contract

To use the Flow Token contract as is, you need to follow these steps:

1. Deploy the `FungibleToken` definition to account `0x02`
2. Deploy the `FlowToken` definition to account `0x03`
3. You can use the `get_balance.cdc` or `get_supply.cdc` scripts to read the 
   balance of a user's `Vault` or the total supply of all tokens, respectively.
4. Use the `setupAccount.cdc` on any account to set up the account to be able to
   use `FlowTokens`.
5. Use the `transfer_tokens.cdc` transaction file to send tokens from one user with
   a `Vault` in their account storage to another user with a `Vault` in their account storage.
6. Use the `mint_tokens.cdc` transaction with the admin account to mint new tokens.
7. Use the `burn_tokens.cdc` transaction with the admin account to burn tokens.
8. Use the `create_minter.cdc` transaction to create a new MintandBurn resource
   and store it in a new Admin's account.


# Running Automated Tests

You can find automated tests in the `fungible_token_test.go` file. It uses the transaction templates that are contained in the `fungible_templates.go` file. Currently, these rely on a dependency from a private dapper labs repository to run, so external users will not be able to run them. We are working on making all of this public so anyone can run tests, but haven't completed this work yet.


# Payment ID solution for Custodial Deposits

We have included a simple example of a contract and resource that could
be used by custodial services to be able to accept deposits from their customers.
This is included in `src/contracts/CustodialDeposit.cdc`. The service would deploy the
contract to their account, which stores the special `DepositResource`
into their storage and published a reference, then users could use transactions
like `src/transactions/custodial_deposit.cdc` to deposit their tokens into the account.
Each deposit has to include a payment ID, or tag as we call it in the contract, to
indicate which account it corresponds to. The resource emits an event that the
service can watch for to see which user's account to credit.

To test, if you have already deployed `FungibleToken.cdc` and `FlowToken.cdc` to
accounts 1 and 2, respectively:

1. Deploy `CustodialDeposit.cdc` to account `0x04`
2. Switch to Account `0x03` and submit the `custodial_deposit.cdc` transaction.

## License 

The works in these folders are under the [Unlicense](https://github.com/onflow/flow-NFT/blob/master/LICENSE):

- [src/contracts](https://github.com/onflow/flow-ft/tree/bindata/src/contracts)



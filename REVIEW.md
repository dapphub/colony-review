# Colony Sale Code Review

Prepared by Ryan Casey \<ryan@dapphub.io\> and Rain \<rain@dapphub.io\>.

## Scope

The web application which interacts with the smart contracts is considered out
of scope. This code review is only concerned with the following files:

1. ColonyTokenSale.sol
2. EtherRouter.sol
3. Migrations.sol
4. Ownable.sol
5. Resolver.sol
6. Token.sol

The code above makes use of the `ds-math` and `ds-erc20`
[dappsys](https://github.com/dapphub/dappsys) libraries of smart contracts,
as well as a custom Token contract combining parts of the `DSToken` and
`DSTokenBase` contracts from the `ds-token` library.

## Findings

- Due to the lack of a price feed, the actual soft cap and minimum raise
  amounts will almost certainly differ from the amounts given in the
  requirements. The degree to which the actual raise differs depends on
  ETH volatility. It may be a good idea to communicate in terms of actual ETH
  amounts once the ICO has begun.
- It might be a good idea to replace the hard-coded numbers in
  `claimVestedTokens` with constants.
- The Colony multisig receives all ETH funds immediately, so refunds are a
  trustful process. This may be counter to user expectations, so clear
  communication around this point is encouraged.
- Per the requirements, CLNY cannot be transferred by buyers (it literally is
  not even distributed) until the multisig manually intervenes. This means ICO
  participants must trust that the multisig will give users their CLNY despite
  having already taken possession of their ETH.
- Dappsys components should be upgraded to their latest version.
- Consider replacing `Ownable` with `DSAuth`, for consistency with Dappsys.
- Solidity v0.4.15 or greater should be used to compile the production
  contracts, due to a bug in `delegatecall` in earlier versions.

Overall the system is fairly straightforward, well-tested, and should not cause
any trouble.


[Dappsys]: https://dappsys.info
[ds-note]: https://github.com/dapphub/ds-note
[ds-auth]: https://github.com/dapphub/ds-auth
[ds-math]: https://github.com/dapphub/ds-math
[ds-token]: https://github.com/dapphub/ds-token
[dappsys-monolithic]: https://github.com/dapphub/dappsys-monolithic

## Usage of Dappsys components

There are two [Dappsys] packages used here: [ds-math] and [ds-token]. We will
comment on the usage of both of these. First, we will comment on [ds-auth],
which is not used but easily could be.

### DS-Auth

The reviewed contracts make use of the `Ownable` pattern, which uses the
`onlyOwner` modifier. Only the `owner` can call functions marked by `onlyOwner`
and the `owner` can be updated by `changeOwner`.

By default, DS-Auth provides identical functionality: when inheriting from
`DSAuth`, functions marked with `auth` can only be called by the `owner`, which
can be updated with `setOwner`.

We suggest the use of DS-Auth as it integrates well with the remainder of
Dappsys and allows for easy extension to more complex access patterns. However,
the Ownable pattern does not appear problematic as specifically implemented
here.


### DS-Math

DS-Math is a mixin for simple and overflow-protected math operations.  A variety
of these operations are used in the reviewed contracts.

DS-Math is used via the [dappsys-monolithic] package. This needs updating to
the latest version, as there has been an update to ds-math.

The `finalize` function makes heavy use of DS-Math. A common pattern is to
calculate percentage fractions of an amount, e.g.

```
uint128 earlyInvestorAllocation = wmul(wdiv(totalSupply, 100), 5);
```

Note that DS-Math does not protect against loss of precision when chaining
operations and operation order must still be considered. In the example above,
there is potential for precision loss by performing `wdiv` before `wmul`.
However, this is avoided because the implementation uses literal `100` and `5`,
rather than actual WAD quantities.

We suggest using a decimal fraction literal:

```
uint128 earlyInvestorAllocation = wmul(totalSupply, 0.05 ether);
```

which uses the fact that `1 ether == WAD`.

Alternately, `uint256` could be used instead of wad math, e.g.

```
uint256 earlyInvestorAllocation = div(mul(totalSupply, 5), 100);
```

provided that `uint256` is used as the base numeric type elsewhere. If
this course is followed then note an important difference between `mul`
and `wmul` is that the former truncates whereas the latter rounds.


### DS-Token

`Token.sol` uses the base ERC20 implementation from ds-token, and extends it
with some storage variables and a `mint` function, with access controlled by the
`Ownable` pattern.

The implemented contract appears safe. However, we note that the functionality
is already implemented in ds-token (albeit using ds-auth rather than Ownable).

We note that the re-ordering of storage variables is needed for use with
the `EtherRouter`.

It is possible to write this contract with less duplication of `DSTokenBase`:

```
import "./dappsys/base.sol";
import "./Ownable.sol";


contract Resolved {
    address resolver;
}


contract TokenLike {
    bytes32 public symbol;
    uint256 public decimals;
    bytes32 public name;
}


contract Token is Ownable, Resolved, TokenLike, DSTokenBase(0) {
    function mint(uint128 wad)
    onlyOwner
    {
        _balances[msg.sender] = add(_balances[msg.sender], wad);
        _supply = add(_supply, wad);
    }
}
```

Using `DSToken`, with `DSAuth` as the ownership pattern, this can be reduced
further:

```
import "./dappsys/token.sol";

contract Resolved {
    address resolver;
}

contract Token is Resolved, DSToken {}
```


## Code Style

### Readability

The large amount of comments and storage variables are difficult to parse.
Consider whether the comments are redundant or could be simplified.

There is a mix between British English 's' and American English 'z' in variable,
event, modifier and function names. Please standardize on one of these.

### Modifiers

There are a number of assertive modifiers defined in the core sale contract.
The `nonZeroAddress` modifier is a nice feature and we note that
https://github.com/ethereum/solidity/issues/2621 is fixed as of solc v0.4.14.

Some of these modifiers could be combined together, for example `saleNotStopped`
could be folded into `saleOpen`.

A pattern to consider here is using a single `canFunctionName` modifier per
function, which lays out all of the call requirements in one place, for example

```
modifier canBuy {
    assert(block.number >= startBlock);
    assert(block.number < endBlock);
    assert (!saleStopped);
    require(msg.value >= MIN_CONTRIBUTION);
    _
}
```

However, this is largely a matter of taste.


### Event naming

It is common practice in Solidity development to write events as `EventName`,
and this convention is followed here. However, we would advise prepending event
names with `Log`, e.g. `LogEventName`. This makes it very clear to the reader
when an event is being emitted.

Further, we would consider annotating functions with the `note` modifier from
[ds-note], making the call history of the contract explicitly clear in the logs.

### Arguments

`_owner` is used several times as an argument name, which is easily
confused with the `owner` variable. `_user` would be a more descriptive
and less confusing name choice.

### Data Structures

The code could be simplified and made more readable by folding the `tokenGrants`
mapping into the `GrantClaimTotal` struct, and renaming the latter to `Grant`.

## Test Coverage

Based on the output of the `gulp test:contracts:coverage` command, all lines of
code are covered with tests. There are 82 tests in total. The tests use the
Truffle framework and are written in Javascript, which can be somewhat dangerous
due to [the potential difficulties in passing data between the
two](https://blog.rexmls.com/the-solution-a2eddbda1a5d). A combination of
Solidity and Javascript tests, or even just Solidity tests alone, would have
been a little better. Still, the level of test coverage is in itself quite
impressive and bodes well for this project.


## Requirements Overview

These are the requirements as collected from [the Colony issue tracker](
https://github.com/JoinColony/colonySale/issues?q=is%3Aissue).

1. <a name="req1">Token contract must be upgradable _in-place_. I.e., a multisig
   will be able to arbitrarily change the behavior of the contract at the token
   address.

   [Reference](https://github.com/JoinColony/colonySale/issues/1)
   </a>
2. <a name="req2">The ICO begins at a particular point in time, defined by a
   block number, and ends at the earlier of:
    - 14 days after the start
    - min(24 hours, max(3 hours, time-taken-to-reach-soft-cap)) + start-time

    [Reference](https://github.com/JoinColony/colonySale/issues/2#issue-227588904)
   </a>
3. <a name="req3">CLNY price set at 0.001 ETH, miniumum purchase size of 0.01
   ETH. (I.e., no buying less than 10 CLNY.)

   [Reference](https://github.com/JoinColony/colonySale/issues/3)
   </a>
4. <a name="req4">15 million USD soft cap, based on exchange rate set at time of
   contract deployment.

   [Reference](https://github.com/JoinColony/colonySale/issues/4)
   </a>
5. <a name="req5">Ability to pause and resume the ICO at any time.

   [Reference](https://github.com/JoinColony/colonySale/issues/5)
   </a>
6. <a name="req6">49% of the CLNY tokens are to be retained and divided among
   early investors (5%), the Colony team (10%), the Colony Foundation (15%), and
   the Colony strategy fund (19%). The Foundation and team shares are to be
   subject to vesting over a 24 month period with a 6 month cliff.

   [Reference](https://github.com/JoinColony/colonySale/issues/9)
   </a>
7. <a name="req7">CLNY must be manually unlocked in order to be transferrable.

   [Reference](https://github.com/JoinColony/colonySale/issues/7)
   </a>
8. <a name="req8">Allow for refunds if less than 5 million USD is raised.

   [Reference](https://github.com/JoinColony/colonySale/issues/10)
   </a>


## Migrations.sol

Contains a contract called `Migrations` which is only used by Truffle for its
own purposes. As this contract is not in any way part of the system under
review, we are ignoring it.


## Ownable.sol

Contains a contract called `Ownable` which is meant to be used as a mix-in. It
provides a `public address owner` property which gets automatically set to
`msg.sender`. It provides an `onlyOwner` modifier which `require`s
`msg.sender` to be `owner` before proceeding. It also provides a `changeOwner`
function which can only be called by the current owner. It takes an address
other than `0x0` and sets `owner` to it.

This is a short and simple contract which does not appear to have any problems.


## Resolver.sol

Contains a contract called `Resolver` which maintains a mapping, `pointers`, of
`bytes4` function signatures to addresses and return value byte sizes (in most
cases 32 bytes, in one case 0 bytes). It provides a `destination` function which
takes a `bytes4` function signature and returns the address associated with the
signature, an `outsize` function which takes the same argument and returns the
`uint` return value size in bytes, a `lookup` function which again takes the
same argument and returns the previous two values as a tuple (`(destination,
outsize)`), and a `stringToSig` function which takes a function signature as a
`string` and returns the `bytes4` signature.

Note that the `bytes4` value used to look up "pointers" are identical to the
values used by Solidiy to look up function calls, and that collisions between
function names are not allowed by the Solidity compiler. Thus collisions should
not pose a threat as long as the `Resolver` is used to forward function calls to
a single contract type compiled with Solidity. Furthermore, all the forwarded
functions in `Resolver` are hard-coded and correspond to ERC20 functions, so
collisions are not even a remote possibility in this case.

This is a short and simple contract which does not appear to have any problems.


## EtherRouter.sol

Contains a contract called `EtherRouter` which inherits from `Ownable`. This
contract wraps a `Resolver` and allows the user to interact with the
`EtherRouter` contract as if it were an ERC20 contract, forwarding calls to the
addresses contained in the `Resolver`'s `pointers` map. It also provides public
`symbol`, `decimals`, and `name` properties which are used by some clients to
improve the user's experience when adding new tokens, and a public `resolver`
property which holds the address of the `Resolver` currently wrapped.

It additionally defines a function called `setResolver` which takes the address
of a `Resolver` and sets the `resolver` property to it. This function may only
be called by the contract's `owner`.

This is the top-level contract which allows in-place upgrading, in accordance
with [requirement #1](#req1). It uses a few lines of inline assembly, but is
otherwise straightforward. It does not appear to pose any problems.

This contract uses `delegatecall`, which is subject to a recent bug in the
Solidity compiler: `DelegateCallReturnValue`. **As a precaution we strongly
recommend deploying the production contracts with solc v0.4.15 or later**.


### Alternative Architecture

We will briefly mention an alternative to an upgradeable token here, whilst
understanding the business requirements that have led to the current design.

Dapphub considers `DSToken` to be a 'box' - that is, a well defined and isolated
component that should not be extended or overridden. Excepting formal
specification and/or direct bytecode implementation we see `DSToken` as very
mature. However, if an upgrade were needed for some reason `DSToken` provides
`stop` and `start`, allowing for the blocking of stateful operations. A
distributor would then be used to initialize balances in the new token.

Persistent addressing would be solved by a solution like ENS, e.g.
`colony-token.eth` would resolve to the current token address.


## Token.sol

Contains a contract called `Token`, which is an extension of the `DSTokenBase`
contract with the `mint` function from `DSToken`, modified to use `Ownable`
rather than `DSAuth`, some standard token attributes, and an `address resolver`
attribute.

We recommend simplifying this contract to avoid repeating `DSTokenBase`, as
noted above.


## ColonyTokenSale.sol

Contains a contract called `ColonyTokenSale`, which encapsulates the ICO logic
and most of the requirements laid out in this document. It inherits from
`DSMath`, giving it access to overflow-protected math operations, and it wraps a
`Token` contract.

The usage of `DSMath` is commented on above.

### Requirements Satisfaction

`ColonyTokenSale` provides the following public properties:

- `uint startBlock`: The block at which the ICO begins.

  See [requirement #2](#req2).
- `uint endBlock`: The block at which the ICO ends.

  See [requirement #2](#req2).
- `uint postSoftCapMinBlocks`:  The minimum number of blocks to wait after the
  soft cap is reached.

  See [requirement #2](#req2).
- `uint postSoftCapMaxBlocks`:  The maximum number of blocks to wait after the
  soft cap is reached.

  See [requirement #2](#req2).)
- `uint constant TOKEN_PRICE_MULTIPLIER = 1000`: Number of tokens per ETH.

  See [requirement #3](#req3).
- `uint constant MIN_CONTRIBUTION = 10 finney`: Minimum purchase size.

  See [requirement #3](#req3).
- `uint minToRaise`: Minimum ICO raise.

  See [requirement #8](#req8).
- `uint public totalRaised`: Total ether raised, plus 1 finney for the purpose
  of not requiring the first buyer to pay extra gas.
- `uint public softCap`: See requirements [#4](#req4) and [#2](#req2).
- `address colonyMultisig`: The address which can call functions modified with
  `onlyColonyMultisig`. Related to claiming raised funds after ICO finalization,
  as well as [requirement #5](#req5).
- `Token token`: Address of the token being sold in this ICO.
- `bool saleStopped`: `true` if the ICO has been paused, `false` otherwise.

  See [requirement #5](#req5).
- `bool saleFinalized`: `true` if the ICO has been finalized, `false` otherwise.

  See [requirement #7](#req7).
- `uint saleFinalisedTime`: The time at which the ICO was finalized.

  See requirements [#6](#req6) and [#7](#req7).
- `address INVESTOR_1`: See [requirement #6](#req6).
- `address INVESTOR_2`: See [requirement #6](#req6).
- `address TEAM_MEMBER_1`: See [requirement #6](#req6).
- `address TEAM_MULTISIG`: Takes the remainder of the development team
  allocation after distributions to team members #1 and #2.

  See [requirement #6](#req6).
- `address FOUNDATION`: See [requirement #6](#req6).
- `address STRATEGY_FUND`: See [requirement #6](#req6).
- `uint128 constant ALLOCATION_TEAM_MEMBER_ONE = 30 * 10 ** 18`: Amount in CLNY
  wei granted to team member #1.

  See [requirement #6](#req6).
- `uint128 constant ALLOCATION_TEAM_MEMBER_TWO = 80 * 10 ** 18`: Amount in CLNY
  wei granted to team member #2.

  See [requirement #6](#req6).
- `uint128 constant ALLOCATION_TEAM_MEMBERS_TOTAL = 110 * 10 ** 18`: Amount in
  CLNY wei granted to both team members.

  See [requirement #6](#req6).
- `mapping (address => uint) public userBuys`: Keeps track of ether spent by
  each participant.
- `mapping (address => uint) public tokenGrants`: Keeps track of tokens granted
  to investors, development team members, the Colony Foundation, and the
  strategy fund.

  See [requirement #6](#req6).
- `mapping (address => GrantClaimTotal) public grantClaimTotals`: Keeps track of
  tokens still unclaimed by investors, development team members, the Colony
  Foundation, and the strategy fund.

  See [requirement #6](#req6).

The constructor takes eight arguments:

- `uint _startBlock`: Sets `startBlock`.
- `uint _minToRaise`: Sets `minToRaise`.
- `uint _softCap`: Sets `softCap`.
- `uint _postSoftCapMinBlocks`: Sets `postSoftCapMinBlocks`
- `uint _postSoftCapMaxBlocks`: Sets `postSoftCapMaxBlocks`
- `uint _maxSaleDurationBlocks`: Sets `endBlock` by addition to `_startBlock`.
- `address _token`: The address of the token to sell. Sets `token`.
- `address _colonyMultisig`: Sets `colonyMultisig`.

The constructor checks the addresses to ensure they aren't set to zero. It uses
a modifier for only one of these checks owing to [a Solidity
bug](https://github.com/ethereum/solidity/issues/2621). This bug has been
resolved recently, so our recommendation is to upgrade to Solidity 0.4.14 and
use the `nonZeroAddress` modifier for both checks.

`_postSoftCapMinBlocks` must be non-zero and must also be less than
`_postSoftCapMaxBlocks`. `_startBlock` cannot be equal to or less than the
current block number.

The contract's fallback function is `payable` and triggers the `internal buy`
function. Thus participating in the ICO only requires sending ETH to the ICO
contract.

The `buy` function cannot be called by other addresses, and it cannot be called
while the sale is paused, while outside the ICO block range (I.e., not until
`startBlock` <= `block.number` < `endBlock`), or with a value of less than 10
finney. It forwards received funds to `colonyMultisig` and adds the amount sent
to the user's total amount sent to date and to the total amount raised overall.
If the soft cap has been reached with the new contribution, it updates
`endBlock` in accordance with [requirement #2](#req2).

The `claimPurchase` function can only be called by `colonyMultisig` after the
sale has been finalized. It takes an address as an argument, `_owner`,
calculates how much CLNY is owed `_owner` based on `TOKEN_PRICE_MULTIPLIER` and
the amount recorded in `userBuys`, then sends the tokens to the user and zeroes
out the `_owner` entry in `userBuys`. This puts the multisig in charge of
distributing tokens to users after the crowdsale, which satisfies [requirement
#7](#req7).

The `claimVestedTokens` does not take any arguments. It allows the transaction
sender to claim any token grants they may be entitled to in accordance with
[requirement #6](#req6) after the ICO has been finalized. It `asserts` that it
has been six months since the ICO was finalized, calculates how much CLNY is
owed the caller based on how much time has passed and how much they've already
claimed, and then sends that amount to the caller.

The `finalize` function may only be called after the ICO sale period has passed,
if the minimum required amount has been raised, and if the ICO hasn't already
been finalized. It calculates how much CLNY to mint based on how much ETH was
raised, in accordance with [requirement #6](#req6). I.e., `totalRaised *
TOKEN_PRICE_MULTIPLIER * 100 / 51`.

The finalize funtion then hands off ownership of the contract to
`colonyMultisig`, immediately transfers the required amount of CLNY to the
investors, team members 1 & 2, and the strategy fund, and then allocates the
vesting grants to the rest of the development team's multisig and the
foundation. Note that the strategy fund does not technically receive 19%, but
instead receives the remainder of the 49% set aside after all the other
allocations and transfers have been made. This helps prevent any trouble that
might have otherwise been caused by mixing percentage-based allocations with
static allocations.

The `stop` and start functions may only be called by `colonyMultisig`. These
pause and resume the ICO. Note that pausing the ICO does not extend its length
at all.


### Review

The `finalize` logic can be given as

```
pS = tR . 1000
tS = pS . 100 / 51

eIA = tS .  5 / 100
tTA = tS . 10 / 100
fA  = tS . 15 / 100

tRA = tTA - 110 . 10^18
sFA = tS - (eIA + tTA + fA + pS)

mint(tS)
push(iN1, eIA)
push(tM1, 30 . 10^18)
push(tM2, 80 . 10^18)
push(sF, sFA)

tG[tM] = tRA
tG[f]  = fA
```

where camelCase variable names are contracted to single letter abbreviations,
`mint => token.mint` and `push => token.transfer`.

Any off-by-one-wei division rounding errors are compensated for by the
definition of `sFA`, which analytically evaluates to `sFA = tS . 19 / 100`, but
could be off from this by a small amount.

DS-Math unsigned subtraction will revert if the result would overflow. This
could occur in the evaluation of `tRA`, i.e. if `tTA < 110 . 10^18`. Working
backwards we obtain `tR < 0.561 . 10^18`, i.e. **`finalize` will revert if the
total raise is less than 0.561 ether**. Owing to the `buy` behaviour of sending
all received ether directly to a standard multisig, the low monetary value of
half an ether and the current ICO climate, we see this as a low severity issue.
However, it is not clear that this code path was anticipated in the contract
design.

Note that this behaviour is related to, but separate from, the `minToRaise`
variable, used by the `raisedMinimumAmount` modifier, which will cause
`finalize` to revert if the total raise is not high enough.


### Data Simplification

The data structures around 'grants' are currently:

```
mapping (address => uint128) public tokenGrants;
struct GrantClaimTotal {
    uint64 monthsClaimed;
    uint128 totalClaimed;
}
mapping (address => GrantClaimTotal) public grantClaimTotals;
```

This could be simplified:

```
struct Grant {
    uint128 totalGranted;
    uint128 totalClaimed;
    uint64  monthsClaimed;
}

mapping (address => Grant) public grants;
```

leading to slightly simpler code. This isn't necessary however, and the public
`tokenGrants` does provide an automatic getter function.


## Assumptions

- The `_maxSaleDurationBlocks` constructor parameter passed to `ColonyTokenSale`
  will correspond to roughly 14 days' worth of blocks.
- The `_postSoftCapMinBlocks` constructor parameter passed to `ColonyTokenSale`
  will correspond to roughly 3 hours' worth of blocks.
- The `_postSoftCapMaxBlocks` constructor parameter passed to `ColonyTokenSale`
  will correspond to roughly 24 hours' worth of blocks.
- The `_minToRaise` constructor parameter passed to `ColonyTokenSale`
  will correspond to roughly 5 million USD worth of ether.
- The `_softCap` constructor parameter passed to `ColonyTokenSale`
  will correspond to roughly 15 million USD worth of ether.

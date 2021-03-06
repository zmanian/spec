# Lite client

A lite client is a process that connects to Tendermint full nodes and then tries to verify application data using the Merkle proofs.

## Context of this document

In order to make sure that full nodes have the incentive to follow the protocol, we have to address the following three Issues

1. The lite client needs a method to verify headers it obtains from full nodes according to trust assumptions -- this document.

2. The lite client must be able to connect to one correct full node to detect and report on failures in the trust assumptions (i.e., conflicting headers) -- a future document.

3. In the event the trust assumption fails (i.e., a lite client is fooled by a conflicting header), the Tendermint fork accountability protocol must account for the evidence -- see #3840

## Problem statement

We assume that the lite client knows a (base) header _inithead_ it trusts (by social consensus or because the lite client has decided to trust the header before). The goal is to check whether another header _newhead_ can be trusted based on the data in _inithead_.

The correctness of the protocol is based on the assumption that _inithead_ was generated by an instance of Tendermint consensus. The term "trusting" above indicates that the correctness on the protocol depends on this assumption. It is in the responsibility of the user that runs the lite client to make sure that the risk of trusting a corrupted/forged _inithead_ is negligible.

## Definitions

### Data structures

In the following, only the details of the data structures needed for this specification are given.

- header fields

  - _height_
  - _bfttime_: the chain time when the header (block) was generated
  - _V_: validator set containing validators for this block.
  - _NextV_: validator set for next block.
  - _commit_: evidence that block with height _height_ - 1 was committed by a set of validators (canonical commit). We will use `signers(commit)` to refer to the set of validators that committed the block.

- signed header fields: contains a header and a _commit_ for the current header; a "seen commit". In the Tendermint consensus the "canonical commit" is stored in header _height_ + 1.

- For each header _h_ it has locally stored, the lite client stores whether
  it trusts _h_. We write _trust(h) = true_, if this is the case.

- Validator fields. We will write a validator as a tuple _(v,p)_ such that
  - _v_ is the identifier (we assume identifiers are unique in each validator set)
  - _p_ is its voting powers

### Functions

For the purpose of this lite client specification, we assume that the Tendermint Full Node exposes the following function over Tendermint RPC:

```go
    func Commit(height int64) (SignedHeader, error)
      // returns signed header: header (with the fields from
      // above) with Commit that include signatures of
      // validators that signed the header


    type SignedHeader struct {
      Header        Header
      Commit        Commit
    }
```

### Definitions

- _tp_: trusting period
- for realtime _t_, the predicate _correct(v,t)_ is true if the validator _v_
  follows the protocol until time _t_ (we will see about recovery later).

### Tendermint Failure Model

If a block _h_ is generated at time _bfttime_ (and this time is stored in the block), then a set of validators that hold more than 2/3 of the voting power in h.Header.NextV is correct until time h.Header.bfttime + tp.

Formally,
\[
\sum*{(v,p) \in h.Header.NextV \wedge correct(v,h.Header.bfttime + tp)} p >
2/3 \sum*{(v,p) \in h.Header.NextV} p
\]

_Assumption_: "correct" is defined w.r.t. realtime (some Newtonian global notion of time, i.e., wall time), while _bfttime_ corresponds to the reading of the local clock of a validator (how this time is computed may change when the Tendermint consensus is modified). In this note, we assume that all clocks are synchronized to realtime. We can make this more precise eventually (incorporating clock drift, accuracy, precision, etc.). Right now, we consider this assumption sufficient, as clock synchronization (under NTP) is in the order of milliseconds and _tp_ is in the order of weeks.

_Remark_: This failure model might change to a hybrid version that takes heights into account in the future.

The specification in this document considers an implementation of the lite client under this assumption. Issues like _counter-factual signing_ and _fork accountability_ and _evidence submission_ are mechanisms that justify this assumption by incentivizing validators to follow the protocol.
If they don't, and we have more that 1/3 faults, safety may be violated. Our approach then is to _detect_ these cases (after the fact), and take suitable repair actions (automatic and social). This is discussed in an upcoming document on "Fork accountability". (These safety violations include the lite client wrongly trusting a header, a fork in the blockchain, etc.)

## Lite Client Trusting Spec

The lite client communicates with a full node and learns new headers. The goal is to locally decide whether to trust a header. Our implementation needs to ensure the following two properties:

- Lite Client Completeness: If header _h_ was correctly generated by an instance of Tendermint consensus (and its age is less than the trusting period), then the lite client should eventually set _trust(h)_ to true.

- Lite Client Accuracy: If header _h_ was _not generated_ by an instance of Tendermint consensus, then the lite client should never set _trust(h)_ to true.

_Remark_: If in the course of the computation, the lite client obtains certainty that some headers were forged by adversaries (that is were not generated by an instance of Tendermint consensus), it may submit (a subset of) the headers it has seen as evidence of misbehavior.

_Remark_: In Completeness we use "eventually", while in practice _trust(h)_ should be set to true before _h.Header.bfttime + tp_. If not, the block cannot be trusted because it is too old.

_Remark_: If a header _h_ is marked with _trust(h)_, but it is too old (its bfttime is more than _tp_ ago), then the lite client should set _trust(h)_ to false again.

_Assumption_: Initially, the lite client has a header _inithead_ that it trusts correctly, that is, _inithead_ was correctly generated by the Tendermint consensus.

To reason about the correctness, we may prove the following invariant.

_Verification Condition: Lite Client Invariant._
For each lite client _l_ and each header _h_:
if _l_ has set _trust(h) = true_,
then validators that are correct until time _h.Header.bfttime + tp_ have more than two thirds of the voting power in _h.Header.NextV_.

Formally,
\[
\sum*{(v,p) \in h.Header.NextV \wedge correct(v,h.Header.bfttime + tp)} p >
2/3 \sum*{(v,p) \in h.Header.NextV} p
\]

_Remark._ To prove the invariant, we will have to prove that the lite client only trusts headers that were correctly generated by Tendermint consensus, then the formula above follows from the Tendermint failure model.

## High Level Solution

Upon initialization, the lite client is given a header _inithead_ it trusts (by
social consensus). It is assumed that _inithead_ satisfies the lite client invariant. (If _inithead_ has been correctly generated by Tendermint consensus, the invariant follows from the Tendermint Failure Model.)

When a lite clients sees a signed new header _snh_, it has to decide whether to trust the new
header. Trust can be obtained by (possibly) the combination of three methods.

1. **Uninterrupted sequence of proof.** If a block is appended to the chain, where the last block
   is trusted (and properly committed by the old validator set in the next block),
   and the new block contains a new validator set, the new block is trusted if the lite client knows all headers in the prefix.
   Intuitively, a trusted validator set is assumed to only chose a new validator set that will obey the Tendermint Failure Model.

2. **Trusting period.** Based on a trusted block _h_, and the lite client
   invariant, which ensures the fault assumption during the trusting period, we can check whether at least one validator, that has been continuously correct from _h.Header.bfttime_ until now, has signed _snh_.
   If this is the case, similarly to above, the chosen validator set in _snh_ does not violate the Tendermint Failure Model.

3. **Bisection.** If a check according to the trusting period fails, the lite client can try to obtain a header _hp_ whose height lies between _h_ and _snh_ in order to check whether _h_ can be used to get trust for _hp_, and _hp_ can be used to get trust for _snh_. If this is the case we can trust _snh_; if not, we may continue recursively.

## How to use it

We consider the following use case:
the lite client wants to verify a header for some given height _k_. Thus:

- it requests the signed header for height _k_ from a full node
- it tries to verify this header with the methods described here.

This can be used in several settings:

- someone tells the lite client that application data that is relevant for it can be read in the block of height _k_.
- the lite clients wants the latest state. It asks a full node for the current height, and uses the response for _k_.

## Details

_Assumptions_

1. _tp < unbonding period_.
2. _snh.Header.bfttime < now_
3. _snh.Header.bfttime < h.Header.bfttime+tp_
4. _trust(h)=true_

**Observation 1.** If _h.Header.bfttime + tp > now_, we trust the old
validator set _h.Header.NextV_.

When we say we trust _h.Header.NextV_ we do _not_ trust that each individual validator in _h.Header.NextV_ is correct, but we only trust the fact that at most 1/3 of them are faulty (more precisely, the faulty ones have at most 1/3 of the total voting power).

### Functions

The function _Bisection_ checks whether to trust header _h2_ based on the trusted header _h1_. It does so by calling
the function _CheckSupport_ in the process of
bisection/recursion. _CheckSupport_ implements the trusted period method and, for two adjacent headers (in term of heights), it checks uninterrupted sequence of proof.

_Assumption_: In the following, we assume that _h2.Header.height > h1.Header.height_. We will quickly discuss the other case in the next section.

We consider the following set-up:

- the lite client communicates with one full node
- the lite client locally stores all the signed headers it obtained (trusted or not). In the pseudo code below we write _Store(header)_ for this.
- If _Bisection_ returns _false_, then the lite client has seen a forged header.

  - However, it does not know which header(s) is/are the problematic one(s).
  - In this case, the lite client can submit (some of) the headers it has seen as evidence. As the lite client communicates with one full node only when executing Bisection, there are two cases
    - the full node is faulty
    - the full node is correct and there was a fork in Tendermint consensus. Header _h1_ is from a different branch than the one taken by the full node. This case is not focus of this document, but will be treated in the document on fork accountability.

- the lite client must retry to retrieve correct headers from another full node
  - it picks a new full node
  - it restarts _Bisection_
  - there might be optimizations; a lite client may not need to call _Commit(k)_, for a height _k_ for which it already has a signed header it trusts.
  - how to make sure that a lite client can communicate with a correct full node will be the focus of a separate document (recall Issue 3 from "Context of this document").

**Auxiliary Functions.** We will use the function `votingpower_in(V1,V2)` to compute the voting power the validators in set V1 have according to their voting power in set V2;
we will write `totalVotingPower(V)` for `votingpower_in(V,V)`, which returns the total voting power in V.
We further use the function `signers(Commit)` that returns the set of validators that signed the Commit.

**CheckSupport.** The following function checks whether we can trust the header h2 based on header h1 following the trusting period method.

```go
  func CheckSupport(h1,h2,trustlevel) bool {
    if h1.Header.bfttime + tp < now { // Observation 1
      return false // old header was once trusted but it is expired
    }
    vp_all := totalVotingPower(h1.Header.NextV)
      // total sum of voting power of validators in h2

    if h2.Header.height == h1.Header.height + 1 {
      // specific check for adjacent headers; everything must be
      // properly signed.
      // also check that h2.Header.V == h1.Header.NextV
      // Plus the following check that 2/3 of the voting power
      // in h1 signed h2
      return (votingpower_in(signers(h2.Commit),h1.Header.NextV) >
              2/3 * vp_all)
        // signing validators are more than two third in h1.
    }

    return (votingpower_in(signers(h2.Commit),h1.Header.NextV) >
            max(1/3,trustlevel) * vp_all)
      // get validators in h1 that signed h2
      // sum of voting powers in h1 of
      // validators that signed h2
      // is more than a third in h1
  }
```

_Remark_: Basic header verification must be done for _h2_. Similar checks are done in:  
 https://github.com/tendermint/tendermint/blob/master/types/validator_set.go#L591-L633

_Remark_: There are some sanity checks which are not in the code:
_h2.Header.height > h1.Header.height_ and _h2.Header.bfttime > h1.Header.bfttime_ and _h2.Header.bfttime < now_.

_Remark_: `return (votingpower_in(signers(h2.Commit),h1.Header.NextV) > max(1/3,trustlevel) * vp_all)` may return false even if _h2_ was properly generated by Tendermint consensus in the case of big changes in the validator sets. However, the check `return (votingpower_in(signers(h2.Commit),h1.Header.NextV) > 2/3 * vp_all)` must return true if _h1_ and _h2_ were generated by Tendermint consensus.

_Remark_: The 1/3 check differs from a previously proposed method that was based on intersecting validator sets and checking that the new validator set contains "enough" correct validators. We found that the old check is not suited for realistic changes in the validator sets. The new method is not only based on cardinalities, but also exploits that we can trust what is signed by a correct validator (i.e., signed by more than 1/3 of the voting power).

_Correctness arguments_

Towards Lite Client Accuracy:

- Assume by contradiction that _h2_ was not generated correctly and the lite client sets trust to true because _CheckSupport_ returns true.
- h1 is trusted and sufficiently new
- by Tendermint Fault Model, less than 1/3 of voting power held by faulty validators => at least one correct validator _v_ has signed _h2_.
- as _v_ is correct up to now, it followed the Tendermint consensus protocol at least up to signing _h2_ => _h2_ was correctly generated, we arrive at the required contradiction.

Towards Lite Client Completeness:

- The check is successful if sufficiently many validators of _h1_ are still validators in _h2_ and signed _h2_.
- If _h2.Header.height = h1.Header.height + 1_, and both headers were generated correctly, the test passes

_Verification Condition:_ We may need a Tendermint invariant stating that if _h2.Header.height = h1.Header.height + 1_ then _signers(h2.Commit) \subseteq h1.Header.NextV_.

_Remark_: The variable _trustlevel_ can be used if the user believes that relying on one correct validator is not sufficient. However, in case of (frequent) changes in the validator set, the higher the _trustlevel_ is chosen, the more unlikely it becomes that CheckSupport returns true for non-adjacent headers.

**Bisection.** The following function uses CheckSupport in a recursion to find intermediate headers that allow to establish a sequence of trust.

```go
func Bisection(h1,h2,trustlevel) bool{
  if CheckSupport(h1,h2,trustlevel) {
    return true
  }
  if h2.Header.height == h1.Header.height + 1 {
    // we have adjacent headers that are not matching (failed
    // the CheckSupport)
    // we could submit evidence here
    return false
  }
  pivot := (h1.Header.height + h2.Header.height) / 2
  hp := Commit(pivot)
    // ask a full node for header of height pivot
  Store(hp)
    // store header hp locally
  if Bisection(h1,hp,trustlevel) {
    // only check right branch if hp is trusted
    // (otherwise a lot of unnecessary computation may be done)
    return Bisection(hp,h2,trustlevel)
  }
  else {
    return false
  }
}
```

_Correctness arguments (sketch)_

Lite Client Accuracy:

- Assume by contradiction that _h2_ was not generated correctly and the lite client sets trust to true because Bisection returns true.
- Bisection returns true only if all calls to CheckSupport in the recursion return true.
- Thus we have a sequence of headers that all satisfied the CheckSupport
- again a contradiction

Lite Client Completeness:

This is only ensured if upon _Commit(pivot)_ the lite client is always provided with a correctly generated header.

_Stalling_

With Bisection, a faulty full node could stall a lite client by creating a long sequence of headers that are queried one-by-one by the lite client and look OK, before the lite client eventually detects a problem. There are several ways to address this:

- Each call to `Commit` could be issued to a different full node
- Instead of querying header by header, the lite client tells a full node which header it trusts, and the height of the header it needs. The full node responds with the header along with a proof consisting of intermediate headers that the light client can use to verify. Roughly, Bisection would then be executed at the full node.
- We may set a timeout how long bisection may take.

### The case _h2.Header.height < h1.Header.height_

In the use case where someone tells the lite client that application data that is relevant for it can be read in the block of height _k_ and the lite client trusts a more recent header, we can use the hashes to verify headers "down the chain." That is, we iterate down the heights and check the hashes in each step.

_Remark._ For the case were the lite client trusts two headers _i_ and _j_ with _i < k < j_, we should discuss/experiment whether the forward or the backward method is more effective.

```go
func Backwards(h1,h2) bool {
  assert (h2.Header.height < h1.Header.height)
  old := h1
  for i := h1.Header.height - 1; i > h2.Header.height; i-- {
    new := Commit(i)
    Store(new)
    if (hash(new) != old.Header.hash) {
      return false
    }
    old := new
  }
  return (hash(h2) == old.Header.hash)
 }
```

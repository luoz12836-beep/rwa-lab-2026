# STUDENT-QUESTIONS.md — Discussion questions (goes inside your submission zip)

Answer directly under each question. 150–300 words each — **reasoning over length**.

The sections follow the four planks of the bridge. There are no standard answers; these are the
real point of the lab.

---

## A. Custody — where the asset actually is (plank a)

**A1.** In Lab 1, `totalCollateral()` read the vault's own ERC-20 balance, and the chain could
prove the invariant. Here, `MockTBillCustodian.realHoldings()` is just a number that a permissioned
address can move. Write the equivalent of `totalCollateral() == totalSupply()` for *this* system.
What does it actually assert, and what does it no longer prove?

> The closest thing to `totalCollateral() == totalSupply()` in this system is
> 
> ```
> custodian.realHoldings() + vault.reserveBalance() >= vault.totalClaimValue()
> ```
> 
> and it is a much weaker statement than Lab 1's. Strict equality cannot be the form here, because the vault deliberately keeps a cash buffer alongside the custodian's holdings. The only readable claim is that the two together cover the shares.
> 
> In Lab 1 both sides were facts the EVM could read for itself. `totalCollateral()` called `balanceOf` on the collateral token, so the number came out of that token's own storage and nobody could move it without moving tokens. Here the left side is `treasury`, a plain `uint256` on `MockTBillCustodian` that `recordPurchase()` adds to and `recordSale()` subtracts from, both behind `CUSTODIAN_ROLE`. It is a number about T-Bills that contains no T-Bill. The right side is `totalSupply * navPerShare / 1e20`, and `navPerShare` is whatever the reporter last passed to `attest()`.
> 
> So the equation asserts that two numbers typed in by privileged addresses are ordered correctly. It no longer proves that anything exists at a custodian.
> 
> Three details make it worse. Nothing enforces it: no `require` in `TBillVault` or `RedemptionQueue` ever reads `realHoldings()`. The vault never calls `recordSale()`, so when the SPV sells T-Bills to fund a redemption the counter stays where it was, and the books overstate the holdings. And it stops holding as soon as the NAV rises above par, because accrued interest enters the system only through `attest()`; nothing on-chain moves the custodian's number for it.
> 
> Take a 1,000 USDC book at par. After `invest(1_000e6)` the custodian says 1,000e6 and the vault holds 0. One `attest(1.25e8)` later, `totalClaimValue()` is 1,250e6. Left side 1,000, right side 1,250.

<br><br><br>

**A2.** The T-Bills are held by an SPV, a legal entity set up so the fund's assets are bankruptcy-
remote from the issuer. Explain in your own words why a *legal* structure is part of a *technical*
design. If the SPV's custodian goes bankrupt, what does the on-chain token entitle its holder to?

> The legal structure is in the design because the technical part cannot finish the job on its own.
> 
> A token is a claim. A claim needs a debtor who is legally obliged, and no contract can create one. `TBillToken` can enforce that a transfer passes both whitelists, and `RedemptionQueue` can enforce that escrowed shares equal `pendingShares()`. Neither can enforce that 1,000,000 USDC of T-Bills sits in an account at a named custodian. That fact only exists off-chain, so the design has to import it, and the only way to import it is to reduce it to a number plus a signature: `realHoldings()`, written by `CUSTODIAN_ROLE`.
> 
> The SPV is what puts a debtor behind that number. It owns the T-Bills; the issuer does not. So if the issuer fails, its creditors have no route to the fund's assets. That separation is what makes the token's value depend on the T-Bills rather than on the issuer's balance sheet. The same logic runs through the other planks: the whitelist exists because the share is a security, and `attest()` exists because someone is legally accountable for the NAV.
> 
> If the custodian goes bankrupt, the token entitles the holder to nothing on-chain. `transfer` still succeeds and `attest` still moves the NAV, because the chain has no way to notice. The failure only shows up when the queue tries to settle and `releaseReserves` reverts on `InsufficientReserves`. What the holder actually owns is a claim through the SPV. Whether that claim reaches the T-Bills depends on whether they were held segregated and identified for the SPV, or commingled with the custodian's own book. In the first case holders rank ahead of the custodian's creditors; in the second they join the queue.

<br><br><br>

---

## B. Attestation — how the chain learns the truth (plank b)

**B1.** `attest()` is the only function that moves the NAV, and it is gated by `REPORTER_ROLE`.
Everything downstream — subscription pricing, redemption payouts, both invariants — believes that
number. Name the failure modes of a single trusted reporter, and one mechanism (technical,
financial, or legal) that mitigates each.

> The reporter fails in four ways, and they need different fixes.
> 
> A stolen key is the obvious one. Whoever holds `REPORTER_ROLE` can name any NAV at all. In this repo it is worse than that: the vault constructor hands the admin `REPORTER_ROLE` and `OPERATOR_ROLE` together, so one key can move the NAV and move the cash through `invest`. Split the keys, and cap what a single call can do. Reject any NAV more than 5% away from the previous one, then widen the cap behind a timelock.
> 
> Bribery is the same power without the theft: nobody stole anything, someone paid for the number. Run several independent reporters and take the median, and make the role post a bond that a proven misreport forfeits.
> 
> An honest mistake is likelier than either. `1.25e8` typed as `12.5e8`, or the 8-decimal NAV confused with the 18-decimal share count. `navPerShare()` only rejects `answer <= 0`, so one decimal slip mis-prices every share in the same block. The bounds above catch most of it; a second signature before posting catches the rest.
> 
> Then there is silence. A reporter that goes quiet, or threatens to, leaves subscriptions priced off an old number and hands the operator leverage over anyone who needs out. Add a heartbeat, a fallback reporter, and a rule that blocks `subscribe` and `settle` once the feed is older than the heartbeat.
> 
> None of these fix the hard case. A reporter who is honest, timely, and simply wrong about the asset looks identical on-chain to one who is right.

<br><br><br>

**B2.** A production feed would check whether `updatedAt` is stale; this lab's mock does not. A
**frozen** NAV is a different attack from a **wrong** NAV. Describe how a stale-but-honest NAV can
be exploited by someone who knows the true value has moved.

> The difference matters. A wrong NAV needs someone dishonest. A frozen NAV needs nothing more than a reporter who is late, and the number on chain was true when it was posted.
> 
> Suppose the fund's real NAV has risen to 1.05 and the feed still says 1.00. `subscribe()` prices through `sharesForAssets()`, which reads `navPerShare()`, which reads `latestRoundData()` and checks nothing else. An attacker who can see the market, or who sees the reporter's transactions before they land, pays 1,000 USDC and receives 1,000 shares at the old price. When the reporter catches up, those shares are worth 1,050. The 50 comes out of everyone who was already in.
> 
> The other direction is worse. If the real NAV has fallen to 0.95 and the feed still says 1.00, the attacker enqueues instead. `RedemptionQueue.enqueue()` stores `assetsAtNav(shares, vault.navPerShare())` as `assetsLocked`, and that number is fixed at enqueue. So they lock in a payout the fund cannot afford, and the shortfall lands on whoever is still holding when `settle` runs.
> 
> The fix is a heartbeat check, but not in the vault. `MockPriceFeed.latestRoundData()` returns `updatedAt = block.timestamp` on every call, so the feed reports itself as freshly updated forever. A staleness check written against this mock would always pass. You have to fix the feed.

<br><br><br>

---

## C. Redemption — T+1 does not settle on-chain (plank c)

**C1.** The queue locks the payout at the NAV of **enqueue** time, not **settle** time. If the NAV
rises between the two, who gains and who loses — the redeemer, or the holders who stayed? Is that
the right allocation of risk, and how would you change it?

> The redeemer loses and the holders who stayed gain, and that is not the only asymmetry in the design.
> 
> At enqueue the payout is fixed at `assetsAtNav(shares, navPerShare())`. If the NAV rises from 1.00 to 1.05 before `settle` runs, the queue still pays the old number. The shares are burned at settle, so total supply drops by the redeemer's whole position while the vault gives up only the old, smaller amount. The 5% of appreciation on those shares stays in the fund and is spread over everyone still holding. The redeemer gave up the upside.
> 
> Run it the other way and the redeemer wins. If the NAV falls to 0.95 they are still paid at 1.00, and the shortfall comes out of the stayers. So the locked rate is a bet on the NAV over the settlement window, and the redeemer chooses when to take it. They enqueue before a drop they know is coming, and they stay out before a rise. They only ever lock a price that helps them.
> 
> That is the wrong allocation. Two changes would fix it. Price each request pro rata against the cash that actually settles in its round, so the payout floats with the NAV at settle. And let a redeemer cancel before settlement and take their escrowed shares back, which restores the choice they currently get in one direction only.
> 
> One more thing makes it worse. `settle()` walks the queue in order and stops when the cash runs out, so a request near the back can wait through several rounds at a price locked long before. The older the ticket, the staler the lock.

<br><br><br>

**C2.** In a 2008 money-market fund breaking the buck, and in USDC's 2023 depeg, redemptions were
handled very differently. Compare the two. For a tokenized T-Bill fund, what does it mean to "close
the redemption channel", and what should happen to the queue when it does? Then look at the lever
this lab actually ships: `pause()` (`PAUSER_ROLE`) freezes transfers, minting **and** redemption
together — one switch, no partial setting. Why is that the wrong tool for a liquidity squeeze, and
what would you rather have?

> The two cases differ in what was broken.
> 
> The Reserve Primary Fund held 785 million of Lehman commercial paper when Lehman filed on 15 September 2008. The paper was worth far less than face, so the assets were impaired. It broke the buck to 0.97 and suspended redemptions. Staying open would have let the fastest redeemers take 1.00 out of a pool that no longer held 1.00 per share.
> 
> Circle in March 2023 had 3.3 billion of 40 billion parked at Silicon Valley Bank. The Treasury portfolio behind USDC was fine; the banking access was not. Circle kept redemptions open at 1:1, took a backlog, and cleared it by 15 March. The peg closed on 13 March. A gate there would have manufactured the run it was meant to stop.
> 
> Closing the redemption channel is therefore a solvency tool, not a liquidity one. For a tokenized T-Bill fund it should mean one thing: stop pricing new exits at NAV. It should not mean freezing the token, because transfers still work at a discount. And the queue already waiting must be repriced pro rata at the final NAV, not paid at its locked price, or the head is whole and the tail is wiped.
> 
> `pause()` cannot express any of that. It is one flag in `_update`, so it stops transfers, mints and burns together, and `settle()` reverts with it. The secondary market dies at the exact moment it is the only exit left.
> 
> I would split the switches. A fee or a gate on primary redemption, sized like the SEC's 2010 money-market rules, prices the exit instead of forbidding it, while transfers stay live and settlement goes pro rata. The pause gets a timelock, a sunset, and a key that is not the reporter's.

<br><br><br>

---

## D. Admission — the guest list and the backdoor (plank d)

**D1.** Every balance change in `TBillToken._update` checks **both** endpoints against the
whitelist. That is what stops a sanctioned address from receiving shares — and also what stops a
completely ordinary user who has not finished KYC. Where is the right line for a regulated fund,
and who should be able to move it?

> The line should be drawn per action, not once for the whole token.
> 
> The obligation sits on the issuer: the register must not carry an ineligible or sanctioned holder. A transfer creates a new register entry, so both endpoints have to be checked. The lab gets that right. What it gets wrong is one test for every balance change. Minting is where the regulated relationship is created, so that is where the heavy work belongs: identity, sanctions screening, source of funds, investor category. A secondary transfer only needs to confirm the buyer's status. Burning is a third case: the payout leaves through a bank account with its own checks.
> 
> `isWhitelisted` is a single boolean, and that is the real weakness. It cannot say that an investor may hold but not subscribe, that the address is an omnibus account, or that the holder is a professional investor in one jurisdiction and not another. ERC-3643 models this as an `InvestorStatus` with a code and a country plus a set of claim topics, which is closer to the right shape. Eligibility is a property of the holder, checked continuously, with force-transfer as the remedy.
> 
> On who moves it: not one key. In this repo `COMPLIANCE_ROLE` sits with the same admin that holds `MINTER_ROLE`, `PAUSER_ROLE`, `REPORTER_ROLE` and `OPERATOR_ROLE`, so one address controls the guest list, the NAV, the cash, the pause and the burn. Split it: a compliance key for the list, an operations key for the cash and the queue, a reporter key for the NAV, and an admin multisig that only rotates roles. Adding a holder should need the compliance key alone. Removing one is the destructive direction: a second signature or a timelock, plus a reason code and notice to the holder.

<br><br><br>

**D2.** A regulated fund asks for two levers: freeze an address (`setWhitelisted(addr, false)`) and
burn a balance (`MINTER_ROLE`) — both censorship tools, both justified by the product. But Ex4.3
shows the two **collide**: once an address is off the list, `burn()` reverts as well, because
`_update` guards both endpoints. So a freeze also blocks a seizure. Argue **both** sides of handing
an issuer these powers, then say what safeguards you would add and who should hold each key.

> Both powers are real requirements, and the case for them is not cynical.
> 
> Without a freeze the fund cannot comply. OFAC and similar regimes require a sanctioned holder to be stopped immediately, and if the token can still move, the issuer is participating in the transfer. Without a seizure the fund cannot obey a court order, a receivership, or an estate, all of which need shares moved out of a name that will never sign. And the power protects the other holders: one sanctioned address on the register can make the whole fund uninvestable.
> 
> The case against is that both levers are unilateral, and in this repo they sit with the same key that moves the NAV and the cash. One compromised address can freeze every holder and burn every balance. There is no notice, no reason code, and no appeal. And `burn()` destroys the position rather than seizing it: a sanctioned holder's shares are not the fund's to take, and a force-transfer to a receiver keeps the position traceable if the match was wrong.
> 
> The collision is the concrete bug. A freeze that also blocks `burn()` leaves the issuer able to stop a holder and unable to remedy it, and the frozen shares stay in `totalSupply()`, so `totalClaimValue()` keeps counting value nobody can move. The burn path should not consult the whitelist, because a forced transfer is the enforcement mechanism for the whitelist.
> 
> What I would add: freeze instant and single-key, because it is protective; seizure behind a timelock and a 2-of-3 that includes the transfer agent, held under a separate `RECOVERY_ROLE`. Force-transfer to a receiver address, not burn. A reason code and a legal reference on every call, with notice to the holder, and a circuit breaker on how many addresses one key can freeze per period.

<br><br><br>

---

## E. Tests (Tier 1 required — this is Ex2, Ex3 and Ex4)

Turn the red tests green in `test/exercises/01_BridgeTasks.t.sol` to cover the scenarios below, and
write your test function names here:

| Plank | Scenario                                                             | Your test function name                           |
| ----- | -------------------------------------------------------------------- | ------------------------------------------------- |
| a     | Shares exist while the custodian holds nothing                       | test_Ex2_SharesExistWhileTheCustodianHoldsNothing |
| a     | `realHoldings()` moves with no cash moving                           | test_Ex2_RealHoldingsIsJustANumber                |
| b     | The claim value floats while the share count stays                   | test_Ex3_TheClaimFloats_TheShareCountDoesNot      |
| b     | One `attest` re-prices every holder at once                          | test_Ex3_OneCallMovesTheWholeBook                 |
| d     | A non-whitelisted address cannot receive shares, even by paying      | test_Ex4_NoKyc_NoShares_EvenIfYouPay              |
| d     | A whitelisted holder cannot send shares to a non-whitelisted address | test_Ex4_TransferChecksBothEndpoints              |
| d     | Removing an address freezes that holder — and blocks a burn          | test_Ex4_FreezeBeatsConfiscation                  |

Then write one more scenario you consider **most likely to be exploited** on a tokenized fund, and
say which plank it breaks:

> Trading ahead of the attestation.
> 
> The vault prices subscriptions through `sharesForAssets()` and the queue prices redemptions through `assetsAtNav()`, and both read the same `navPerShare()`. That number is stale by construction: a NAV is struck at a cutoff and posted afterwards, so there is always a window in which the true value has moved and the on-chain value has not. Anyone who can see the direction can trade into it. The administrator's daily file leaks the direction before the chain does, and so does the reporter's pending `attest()` in the mempool. Subscribe before an upward revision and you buy shares below value. Enqueue before a downward one and `assetsLocked` fixes your payout at the higher, older price before anyone can correct it. No key is compromised, no function is wrong, and every transaction is valid.
> 
> That breaks plank (b). Attestation is the bridge's only source of truth, and it is a truth that arrives late.

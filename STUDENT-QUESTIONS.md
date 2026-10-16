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

> Your answer:

At the simplest stage, before redemption requests exist, the comparable relationship is `vault.reserveBalance() + custodian.realHoldings() >= vault.totalClaimValue()`. Both sides must use six-decimal USDC units. Equality describes exact reported backing; the inequality allows a surplus. Unlike Lab 1, share supply alone is not a dollar liability, because each share must be valued at NAV.

Once redemptions exist, accounting must avoid counting escrowed shares twice. A more complete reported liability is `vault.assetsForShares(tBill.totalSupply() - queue.pendingShares()) + queue.pendingAssets() + queue.totalClaimable()`. Compare this with vault cash, queue cash, and reported custodian assets, allowing for rounding. Pending shares receive their locked payout rather than today's NAV; settled shares have already been burned, but their unclaimed cash remains owed.

This relationship asserts consistency between reported assets and recognized claims. It no longer proves that Treasury bills exist, belong to the fund, are unencumbered, or can be sold. An authorized custodian can increase `realHoldings()` without receiving cash. Moreover, that number is not automatically a current market valuation. Reliable backing therefore requires reconciliation, valuation, and independent custody evidence beyond arithmetic on chain.
<br><br><br>

**A2.** The T-Bills are held by an SPV, a legal entity set up so the fund's assets are bankruptcy-
remote from the issuer. Explain in your own words why a *legal* structure is part of a *technical*
design. If the SPV's custodian goes bankrupt, what does the on-chain token entitle its holder to?

> Your answer:

The legal structure determines whether the asset promised by the software can actually be recovered. A token contract can record balances and restrict transfers, but it cannot establish ownership of securities at a bank or instruct a bankruptcy court. An SPV separates the fund's assets and obligations from the issuer's other business, while custody agreements identify who holds the securities and for whose benefit. Those arrangements define the real meaning of the token's redemption promise.

Bankruptcy remoteness is a risk reduction mechanism, not an absolute guarantee. If the custodian fails, properly segregated securities may be recoverable for the SPV rather than available to the custodian's general creditors. The result depends on applicable law, custody arrangements, asset identification, and any actual shortfall. Cash deposits can also have a different legal treatment from custodial securities.

The token holder receives only the contractual or beneficial rights granted by the fund documents. That might mean a claim against the SPV, a distribution through liquidation, or rights exercised through a trustee. It does not automatically mean direct ownership of particular Treasury bills or immediate payment. This lab's contracts themselves create none of those enforceable legal protections.
<br><br><br>

---

## B. Attestation — how the chain learns the truth (plank b)

**B1.** `attest()` is the only function that moves the NAV, and it is gated by `REPORTER_ROLE`.
Everything downstream — subscription pricing, redemption payouts, both invariants — believes that
number. Name the failure modes of a single trusted reporter, and one mechanism (technical,
financial, or legal) that mitigates each.

> Your answer:

A single reporter creates several distinct failure modes. First, its signing key can be stolen, allowing an attacker to publish a profitable false NAV. Hardware signing and approval by independent signers reduce that risk. Second, the reporter can deliberately lie or collude with traders. Independent valuation sources, external reconciliation, and enforceable liability make deception harder and more costly. Third, an honest reporter can submit the wrong units, decimals, or valuation. Schema validation, sanity bounds, and review of unusually large changes reduce operational mistakes. Fourth, an outage or censorship can freeze an otherwise honest price. A freshness limit and an independently authorized replacement process prevent indefinite trading against stale information.

None of these controls proves asset ownership by itself. Several signers using the same incorrect data still produce an incorrect result, and price bounds can reject legitimate market moves. Controls need explicit escalation procedures.

There is also an important implementation exception: although `vault.attest()` checks `REPORTER_ROLE`, `MockPriceFeed.setPrice()` is public and unrestricted. Anyone can bypass the vault and modify the feed directly. A production implementation must restrict that setter before reporter governance provides meaningful protection. The role check alone does not secure the actual price source.
<br><br><br>

**B2.** A production feed would check whether `updatedAt` is stale; this lab's mock does not. A
**frozen** NAV is a different attack from a **wrong** NAV. Describe how a stale-but-honest NAV can
be exploited by someone who knows the true value has moved.

> Your answer:

A stale NAV can be perfectly accurate for its original valuation time while offering unfair trades now. Suppose the published NAV remains one dollar, but reliable information shows that each share is now worth $1.10. An informed investor can subscribe at the old price and receive more shares than the current asset value justifies. When the NAV catches up, that investor captures value diluted from existing holders. Realizing the gain still depends on later redemption or a buyer; the stale quote alone does not create cash.

The opposite direction is also dangerous. If true value falls to $0.90 while the published NAV remains one dollar, an informed holder can enqueue redemption at the old price. The queue locks that excessive payout, so a later correction does not repair the ticket. If paid, the overpayment reduces assets available to remaining investors.

I would reject new subscriptions and redemption pricing after a defined freshness deadline, require a new valuation before reopening, and consider batch pricing to reduce timing advantages. Importantly, this mock returns `block.timestamp` as `updatedAt` on every read. Checking that field alone would falsely indicate freshness; the feed must store the actual valuation or update timestamp.
<br><br><br>

---

## C. Redemption — T+1 does not settle on-chain (plank c)

**C1.** The queue locks the payout at the NAV of **enqueue** time, not **settle** time. If the NAV
rises between the two, who gains and who loses — the redeemer, or the holders who stayed? Is that
the right allocation of risk, and how would you change it?

> Your answer:

Locking the payout converts the redeemer's position from a floating share claim into a fixed cash claim. Suppose someone queues 1,000 shares at a NAV of one dollar. The ticket promises 1,000 USDC. If NAV rises to $1.10 before settlement, the ticket still pays 1,000, so the redeemer gives up the subsequent upside. Economically, holders who remain benefit from any genuine asset gains retained after satisfying that fixed claim. Conversely, if asset values fall, the fixed payout protects the redeemer and shifts more downside onto remaining holders, assuming sufficient funds are ultimately available.

However, changing the reported NAV alone creates no gain or cash. In this implementation, escrowed shares remain in total supply until settlement, so `totalClaimValue()` also continues repricing them. That helper is not a complete measure of liabilities once tickets are locked.

The allocation can be reasonable if disclosed and paired with fair valuation and liquidity policies. Otherwise it encourages early exits before losses are recognized. I would consider forward pricing at a common valuation cutoff, with transaction costs allocated to redeeming investors. Existing fixed tickets should not be repriced retroactively without contractual authority and a transparent process.
<br><br><br>

**C2.** In a 2008 money-market fund breaking the buck, and in USDC's 2023 depeg, redemptions were
handled very differently. Compare the two. For a tokenized T-Bill fund, what does it mean to "close
the redemption channel", and what should happen to the queue when it does? Then look at the lever
this lab actually ships: `pause()` (`PAUSER_ROLE`) freezes transfers, minting **and** redemption
together — one switch, no partial setting. Why is that the wrong tool for a liquidity squeeze, and
what would you rather have?

> Your answer:

The Reserve Primary Fund's 2008 crisis involved a NAV below one dollar and suspended redemption payments, followed by liquidation distributions. The approved plan treated unpaid investors proportionately rather than rewarding their request timing. USDC's March 2023 depeg instead involved uncertainty over bank reserves and interrupted primary redemption access; on-chain trading continued at market prices. Circle subsequently resumed processing and cleared its backlog. These cases distinguish asset losses and liquidation from temporary banking access problems.

For a tokenized fund, closing redemption should specify whether new requests, pricing, settlement, or withdrawals are stopped. Existing tickets and their ordering should remain recorded; a liquidity shortage should not silently delete claims or allow insiders to jump ahead. Insolvency may require a separately authorized restructuring or proportional distribution.

The lab's token pause blocks minting, transfers, enqueue transfers, and settlement burns together. It unnecessarily disables otherwise legitimate activity during a liquidity squeeze. It also does not block `claim()`, which transfers USDC rather than TBILL. I would use separate subscription, enqueue, and settlement controls, preserving funded claims unless a specific emergency justifies stopping them. Reopening conditions and authority should be published.
<br><br><br>

---

## D. Admission — the guest list and the backdoor (plank d)

**D1.** Every balance change in `TBillToken._update` checks **both** endpoints against the
whitelist. That is what stops a sanctioned address from receiving shares — and also what stops a
completely ordinary user who has not finished KYC. Where is the right line for a regulated fund,
and who should be able to move it?

> Your answer:

For a regulated fund, the eligibility boundary should follow its documented investor requirements and applicable restrictions. A wallet should receive shares only after the fund has linked it to a verified eligible investor. Screening both sender and receiver prevents an approved account from transferring the position to an unverified wallet. The same rule also blocks an ordinary applicant whose checks are incomplete; that is an operational consequence that should be explained before the applicant sends money.

The boundary should not depend on an issuer employee's personal preferences. A designated compliance team should administer routine approvals under published criteria, with documented reasons, a review process, and procedures for wallet replacement. Urgent restrictions may need prompt action, but durable policy changes should require broader governance and legal review. Personal identity documents should remain outside the public blockchain; only the necessary eligibility status should be exposed.

Technically, whitelist administration and role administration should use separate credentials and auditable events. Sensitive removals should receive additional approval where practicable. Users need a channel to challenge mistakes, while genuine restrictions need consistent enforcement. Finally, this lab's whitelist does not verify identity itself: it merely enforces the status supplied by an authorized administrator.
<br><br><br>

**D2.** A regulated fund asks for two levers: freeze an address (`setWhitelisted(addr, false)`) and
burn a balance (`MINTER_ROLE`) — both censorship tools, both justified by the product. But Ex4.3
shows the two **collide**: once an address is off the list, `burn()` reverts as well, because
`_update` guards both endpoints. So a freeze also blocks a seizure. Argue **both** sides of handing
an issuer these powers, then say what safeguards you would add and who should hold each key.

> Your answer:

Issuer controls can support a regulated product: freezing can stop unauthorized transfers, while a narrowly defined forced transfer or burn can implement an enforceable order or correct an operational error. Without recovery mechanisms, a stolen wallet could permanently control a legally registered investment. Investors may reasonably prefer supervised intervention to irreversible theft.

The same powers create serious abuse risks. A compromised administrator can exclude legitimate holders, and unrestricted burning can destroy their positions without compensation. MINTER_ROLE is especially powerful here because it authorizes both issuance and burning, so it should not be treated as a harmless settlement permission.

Ex4.3 exposes a policy collision: removing a holder from the whitelist also prevents burning that holder's balance. I would not solve this by removing all whitelist checks. Instead, a production design could separate transfer eligibility from a narrowly authorized seizure workflow, with an identified legal basis, destination, and complete audit record.

Routine eligibility belongs to compliance personnel; exceptional seizure should require independent legal and operational approvals through a multisignature account. Minting, settlement burning, emergency pausing, and role administration should have separate permissions. Nonurgent changes should have delays, and emergency actions should trigger prompt review and notification. These safeguards constrain power without pretending to eliminate trust.
<br><br><br>

---

## E. Tests (Tier 1 required — this is Ex2, Ex3 and Ex4)

Turn the red tests green in `test/exercises/01_BridgeTasks.t.sol` to cover the scenarios below, and
write your test function names here:

| Plank | Scenario | Your test function name |
|---|---|---|
| a | Shares exist while the custodian holds nothing | `test_Ex2_SharesExistWhileTheCustodianHoldsNothing` |
| a | `realHoldings()` moves with no cash moving | `test_Ex2_RealHoldingsIsJustANumber` |
| b | The claim value floats while the share count stays | `test_Ex3_TheClaimFloats_TheShareCountDoesNot` |
| b | One `attest` re-prices every holder at once | `test_Ex3_OneCallMovesTheWholeBook` |
| d | A non-whitelisted address cannot receive shares, even by paying | `test_Ex4_NoKyc_NoShares_EvenIfYouPay` |
| d | A whitelisted holder cannot send shares to a non-whitelisted address | `test_Ex4_TransferChecksBothEndpoints` |
| d | Removing an address freezes that holder — and blocks a burn | `test_Ex4_FreezeBeatsConfiscation` |

Then write one more scenario you consider **most likely to be exploited** on a tokenized fund, and
say which plank it breaks:

> Your answer:

I would prioritize a test called `test_PublicFeedSetterLetsInvestorLockInflatedRedemption`, targeting plank B, attestation. Unlike a hypothetical stolen reporter key, the current mock exposes a concrete authorization gap: any caller can invoke `feed.setPrice()` directly. A KYC-approved investor can therefore manipulate pricing without holding REPORTER_ROLE.

The scenario starts with an honest investor subscribing 1,000 USDC at par. The attacker independently subscribes another 1,000 USDC and receives 1,000 shares. Acting only as the attacker, the test sets the feed to two dollars, enqueues those shares to lock a 2,000 USDC payout, and restores NAV to one dollar. A normally authorized settler then processes the ticket. The attacker claims 2,000 USDC, while the vault is empty and the honest investor still holds shares worth 1,000 USDC at the restored NAV.

Assertions should verify the attacker's lack of reporter permission, the locked payout, successful settlement, attacker profit, and the remaining backing shortfall. The queue's escrow and cash invariants can still pass: they verify accounting, not truthful pricing. This makes authorization of the actual feed setter an essential fix before deploying beyond a classroom demonstration.
<br><br><br>    
# RFC-0001 — Community Layer Economics

**Status:** Draft · v0.3
**Topic:** How peers in the ANTS community exchange work — without money among themselves, on top of a protocol that does not forbid money to others.
**Audience:** You, if you are willing to disagree in writing.
**Depends on:** [RFC-0002 (Semantic Cache)](./RFC-0002-semantic-cache.md), [RFC-0006 (Payment Terms)](./RFC-0006-payment-terms.md)

---

## A Tuesday evening

It's a Tuesday evening. Your laptop has been on for four hours.

While you wrote emails, it ran a process you barely noticed. It served forty-two cache lookups to other peers. It routed two hundred and seventeen query packets. It verified the output of a model running on a server in Helsinki. Six minutes ago, you asked your laptop a question about Italian industrial-construction law. The answer came back in 800 milliseconds — half of it from a cached response signed by a peer in Trento who specialised the domain two years ago, the other half from a fresh inference whose result your laptop has now added to the cache for the next person who needs it.

You did not pay for it.

You did not pay for it because, while you were writing emails, you were already paying.

This document specifies the rules of that exchange.

---

## What this document is, and what it is not

This document specifies the economy of the **ANTS community layer** — the people, machines, and institutions that have adopted barter as their way of using the protocol.

It is *not* the only economy that the ANTS protocol carries. The protocol itself is neutral: it specifies how peers exchange queries, responses, attestations, and cached content. It does not specify *how peers settle with each other*. That is left to the participants.

The community layer is one such settlement model — symmetric, money-free, locally accounted. Commercial operators may serve inference for fiat payment on the same protocol; gift-economy peers may serve for free; hybrid peers may switch modes per query. All coexist. This document is about the *first* economy, not the *only* one.

For the mechanism by which peers declare their pricing terms in protocol-visible metadata, see [RFC-0006 — Payment Terms](./RFC-0006-payment-terms.md). For the cache layer that underpins all economies including this one, see [RFC-0002 — Semantic Cache](./RFC-0002-semantic-cache.md).

---

## What the community layer is trying to avoid

Before we describe what the community layer *does*, three things it explicitly *does not* do — because each is a path most peer-to-peer AI projects have taken, and each ends somewhere we do not want to go.

**The community does not issue a token.** No coin, no stake, no economic instrument that can be bought, sold, dumped, or rugged. The moment a community has a token, half its participants are there for the token and the other half resent them for being there. The protocol's job becomes defending the token's price instead of providing the service. We are not running that experiment.

**The community does not accept money among its members.** Within the community layer, the medium of exchange is *work*. Commercial operators running on the same protocol *may* accept money — that is their right and their burden, specified through Payment Terms — but the community does not. This keeps the community's economy free from regulatory capture, from speculative pressure, and from the slow drift toward central authority that fiat acceptance creates.

**The community does not rely on charity.** Folding@home works because curing cancer is, for many people, more compelling than the cost of the electricity. "Answering a stranger's question about construction law" is not. If the community's only economy is altruism, the supply side collapses the first time someone calculates their power bill.

What remains is *barter built on memory*. You give work, you receive work. The ledger is local. The cache layer multiplies the value of every good answer over time. The price is participation.

---

## The core idea, in plain English

Every peer in the community is both a producer and a consumer at the same time. There is no separate class of "miners" who serve and "users" who request. The same machine, in the same second, does both.

When peer **A** wants something from the network — an inference, a cache lookup, a verified result — the network looks at what **A** has been *giving* recently, and matches the request to peers who have themselves received from **A**, or who are willing to take a chance on **A** because **A** looks like a productive participant.

Nothing more complicated than that. If you have spent five minutes uploading on BitTorrent, you have already used a protocol that works this way.

What is hard is making this work for *artificial intelligence*, where the unit of exchange is not a megabyte but a *cognitive service* — and where cognitive services come in shapes wildly different from each other.

The rest of this document is about that hardness.

---

## The role of the semantic cache

Before getting into ledgers and exchange rates, one point that changes everything else.

In the community layer, **a meaningful fraction of all served work is a cache hit**, not a fresh inference. A peer that produced a high-quality answer to "what is the correct procedure for filing a Sicilian agritourism subsidy?" is not paid once — it is paid *every time* that question, or one near enough in embedding space, is asked again. The answer lives in the distributed cache, signed by the original producer, attested, and amortised over thousands of subsequent retrievals.

This single fact resolves the structural pathology that pure barter would otherwise have. A specialist whose expertise was rare and whose own demands were modest used to accumulate unspendable credit; now their good answers become *durable yield*. A laptop without a GPU is no longer a charity case; hosting cache shards is genuinely valuable, and earns proportionally.

The cache is specified in [RFC-0002](./RFC-0002-semantic-cache.md). The economy specified below assumes it exists. Read that document before you finish this one if you want to know how the gears actually mesh.

---

## The service unit

You cannot barter without a unit. BitTorrent has it easy: a piece is a piece, a megabyte is a megabyte, the hash is the verifier. ANTS has it hard, because a "service" can be:

- a token of language-model output, on a model that may be 7B or 400B parameters;
- a cache hit that saves another peer twelve seconds of GPU time;
- a verifier's signed attestation;
- one hour of holding a 4-gigabyte cache shard at 99% uptime;
- a hop in a query-routing path.

These are not commensurable in the way bytes are. So we do not pretend they are. Instead, the community layer declares a base unit and lets everything else float against it.

**The base unit is the *normalised compute-second* (NCS).** One NCS is the amount of work an idealised reference machine — a vanilla CPU core at 3 GHz, ignoring GPU acceleration — completes in one second. Every other kind of work is priced in NCS by published benchmark, not by self-declaration:

| Service type        | Indicative pricing                       | Who sets it |
|---------------------|------------------------------------------|-------------|
| LLM inference       | NCS per token, per model family          | Verifier consensus, periodic re-benchmark |
| Cache hit           | A fixed fraction of the saved inference cost (typically 5–15%) | Network parameter |
| Cache hosting       | NCS per GB-hour at declared uptime       | Network parameter |
| Verification        | NCS per check, per model size            | Bench against inference cost |
| Routing hop         | NCS per packet, tiny but non-zero        | Bench against round-trip overhead |
| Annotation          | NCS per labelled item, weighted by trust | Negotiated per labelling task |

Two things to underline.

First: **NCS is a unit of measurement, not a currency.** It is not transferable across peers as a balance. It exists only inside the *local* ledger of each peer pair, like a friendship's mental record of who paid for the last round of drinks. Nobody publishes their total. There is no aggregate, no leaderboard, no top-10.

Second: **the price table is not hard-coded.** It updates as the network re-benchmarks itself. If GPUs get four times faster, the NCS-per-token for the same model drops accordingly. The unit is anchored to the CPU core specifically because CPU performance has been the slowest-evolving thing in computing for the last decade — using it as the meter-stick keeps the rest of the table relatively stable.

---

## The local ledger

Every peer holds a small structured record for every other community-layer peer it has interacted with. Not a blockchain. Not a shared store. A plain, local, append-only log. Think SQLite, or a TOML file if you want to be radical.

The record looks roughly like this:

```
peer:  zXr7...8KqA
since: 2026-05-14T09:02
served-to-them:    +1430.5 NCS   (their unpaid balance with us)
served-by-them:    +1267.2 NCS   (our unpaid balance with them)
net-balance:       +163.3 NCS    (we are slightly ahead)
slot-history:      last 240 minutes of interactions
last-quality:      0.97  (rolling average of verified-correct rate)
choked:            false
optimistic-since:  null
```

When peer **A** receives a service request from peer **B**, **A** consults *its own* record for **B**, makes a local decision, and acts. There is no consultation with a third party. There is no global state. There is no "block confirmation".

If **B** has been consistently giving more than receiving with respect to **A**, **A** treats **B** as a priority customer. If **B** has been consistently receiving without giving back, **A** chokes **B** — politely declines further service until balance is restored, or until the optimistic-unchoke loop tries them again. (See cold start below.)

This is the entire heart of the community-layer economy. Almost everything else is parameter-tuning around this loop.

---

## The choke / unchoke algorithm

The unchoke algorithm runs on every peer, every ten seconds. (Ten seconds is what BitTorrent settled on after a decade of empirical tuning; we inherit it as a default and expect it to be revisited.)

Pseudocode, intentionally readable:

```python
def unchoke_round(peers, slots=8):
    """
    Decide which peers we will serve in the next 10 seconds.
    `slots` is the maximum number we can serve in parallel.
    """
    # 1. Rank peers by how generous they have been to us recently.
    ranked = sorted(
        peers,
        key=lambda p: p.served_to_us_last_minutes(window=20),
        reverse=True,
    )

    # 2. Reserve 1 slot in 4 for optimistic unchoke
    #    (try a peer we haven't served recently, give them a chance).
    optimistic_slots = max(1, slots // 4)
    earned_slots = slots - optimistic_slots

    chosen = ranked[:earned_slots]
    chosen += pick_optimistic_candidates(peers, n=optimistic_slots)

    for p in peers:
        p.choked = p not in chosen
```

Three things are deliberate here.

**The window is short** (twenty minutes, configurable). A peer cannot rest on past glory. Two days of generous uploading buys you nothing if you spent the last hour leeching. This punishes drift and keeps the network responsive to current participation rather than historical reputation. Reputation, in the community layer, is *recent behaviour*, not accumulated wealth.

**The optimistic unchoke is non-negotiable.** Even if eight perfect customers are queued, we always reserve a slot for someone we have not served recently. Without this, the network calcifies into established trade routes and newcomers cannot get in. The cost is small (12.5% of capacity at any moment); the benefit is enormous (network porosity).

**Quality matters but does not dominate.** A peer that always returns correct verified output gets weighted up. A peer that gets caught cheating gets weighted down. But quality is a *multiplier* on the generosity ranking, not a replacement for it. We do not want a perfectly honest peer who never gives anything to outrank a slightly imperfect peer who is contributing actively.

---

## Cold start

The hardest moment in any barter network is the first second. A new peer has nothing to give and nothing to claim. How does it bootstrap?

The community layer solves this with two mechanisms working together — and the cache layer makes a third one possible.

**The compute-floor.** A new peer with no specialised capability enters as a *compute provider* — running commodity inference, caching, routing, or verification on its CPU/RAM/storage. None of these require GPU or specialisation. Whatever machine the peer has, it has *something* to offer at the lowest tier. The protocol guarantees that any peer running the reference client can earn NCS, even on a laptop without dedicated graphics hardware, by doing the unglamorous infrastructure work.

**Optimistic unchoke from the other side.** Other peers, by the rule above, dedicate ~12.5% of their service slots to peers they have not served recently. When a new peer makes its first request, it has a non-trivial probability of being optimistically unchoked by *somebody*. The first request gets served on faith. The recipient then either gives back (and is rewarded with future priority), or doesn't (and is choked out as soon as the optimism slot rotates).

**Cache hosting from minute one.** A new peer can immediately host a slice of the distributed cache. Hosting earns NCS continuously, at low rate but reliably, with no specialisation required beyond storage and uptime. This gives newcomers an income floor while they build reputation.

In practice, a new peer that runs the reference client honestly and stays online for 90 minutes will accumulate enough balance to make its first inference request comfortably. We have no field data for this number yet — it is an estimate, not a measurement — and we mark it as one of the parameters we expect to revise.

We *do not* offer a "free credit allowance" for new peers. Free credits, in every system that has tried them, attract abuse at exactly the rate they discourage real contribution. The compute-floor mechanism replaces them: you don't get free anything, but you do get *cheap entry*, because everyone has a CPU.

---

## Cross-role exchange rates

This is one of the most interesting hard problems in the community layer, and the one we are least sure we have right.

The question: when peer **A** gives peer **B** five seconds of GPU inference, and peer **B** has given peer **A** an hour of cache-hit service, how do those two compare?

Three possible answers:

**(a) Fixed table.** Publish, at the protocol level, exact rates: 1 GPU-second = 200 NCS, 1 cache-hit = 12 NCS, etc. Simple, predictable, easy to implement. But fragile: hardware prices change, models change, and a fixed table either over-rewards or under-rewards a category.

**(b) Per-pair negotiation.** Each peer pair maintains its own rates, learned over time. Like two friends who have figured out their own way of splitting bills. Maximally flexible but creates an enormous cold-start problem within every pair, and makes the network non-fungible.

**(c) Floating, network-wide.** Rates are derived from observed average exchange across the entire network, sampled and gossiped. Like a foreign exchange market. Adapts to real conditions, but requires a price-discovery layer, which adds complexity and is itself attackable.

**Our current bet: (a) with periodic re-anchoring.** Start with a fixed published table in the reference client. Every quarter, the network's verification nodes re-benchmark the relative costs and the table updates as part of a coordinated protocol-version bump. This sacrifices responsiveness to short-term shifts, but it is implementable in 2026 with technology we already have, and it keeps the protocol's surface area small.

We expect (c) to become tractable later, once we have a verification layer mature enough to anchor a price-discovery mechanism. RFC-0003 (verification) is a prerequisite for that conversation.

---

## Specialist credit and the cache

This is where the cache layer transforms what would otherwise be a structural problem.

In a pure barter network, a peer with a unique high-value specialisation accumulates credit faster than they can spend it. Their own demands are modest; the network's demands on them are constant. Without a way to convert this credit into something they want, they eventually leave. The network loses its rarest peers.

With a semantic cache as a first-class layer, this resolves itself. The specialist's high-value answer is not consumed once — it lives in the cache, retrievable by embedding similarity, served thousands of times to peers with similar questions. Each retrieval earns the original producer a fraction of the inference cost (the cache-hit fee specified in the price table). One excellent answer becomes a *durable revenue stream* over months and years, not a one-time payment.

A peer who has contributed 50 well-cited canonical answers to the cache is worth more to the network than a peer who runs 50 raw inferences per day. The protocol rewards both, but the cache mechanism makes the specialist's contribution recur indefinitely.

This is the structural answer to the question every barter network has previously failed to solve.

---

## How this layer interacts with other economies

Community-layer peers operate on the same protocol as commercial operators, gift-economy peers, and hybrid participants. They are interoperable in the sense that they all speak the protocol and can route queries to each other. They are *not* fungible economically:

- A community peer **cannot** spend its NCS balance with a commercial peer. The commercial peer wants fiat, declared in its [Payment Terms](./RFC-0006-payment-terms.md). The community peer would need to acquire fiat by other means.
- A commercial peer **may** serve community-layer peers free, or accept a community-tier rate, or refuse altogether. That is its choice.
- A hybrid peer **may** publish multiple Payment Terms simultaneously: "I serve commercial customers at $0.003/1K token, and community peers at the standard NCS rate." It then maintains both a fiat receivable and a local NCS ledger.

The cache layer is shared across economies. A community peer's well-cited answer is retrievable by a commercial peer's customer, and vice versa. The rules around cache retrieval payments (who pays whom when a community-produced answer serves a commercial-paid query) are specified in [RFC-0006](./RFC-0006-payment-terms.md).

The protocol enforces no economic boundary between these populations. The boundary is purely behavioural: each peer chooses whom to serve and how, by declaring its Payment Terms and adhering to them.

---

## Failure modes and how the community layer survives them

A few failure modes, each with the design's response.

**A peer goes offline mid-service.** The local ledger marks the service as incomplete. The requesting peer routes the remainder to another candidate. No NCS is credited for the failed delivery. Repeated failures push the unreliable peer toward choke status.

**A peer returns garbage output.** The verification layer (RFC-0003) catches this, downweights the peer's quality multiplier, and — in egregious cases — flags it on the reputation ledger (RFC-0004). The local choke status changes immediately for the affected pair.

**A peer floods requests, hoping to consume more than it provides.** The local choke/unchoke loop handles this naturally: as soon as the peer's net-balance with us turns negative beyond a threshold, we stop serving. No special anti-DDoS logic needed at the economic layer.

**A pair becomes asymmetric — A keeps wanting things from B, B never wants anything back.** A simply runs out of credit with B and must route requests to peers who *do* want what A offers — or, more commonly, finds the answer in the cache without needing B at all. The network self-organises toward pairs and triangles where exchange is genuinely mutual.

**Sybil attacks via cloud TEEs.** Handled by RFC-0005 (hardware attestation) and RFC-0004 (reputation slashing). Not the economic layer's problem to solve, but worth flagging that the economic layer assumes the identity layer works.

---

## What we have not figured out yet

If you are about to open an issue, here are the places we would most like sharp eyes:

- **The 20-minute window is a guess.** Empirical data from the first month of testnet will tell us if it should be shorter (responsive but noisy) or longer (stable but forgiving).
- **The optimistic-unchoke ratio.** 12.5% is inherited from BitTorrent. AI services have different cost variance than file pieces. The right number might be different.
- **The slot count per peer.** Set at 8 in the pseudocode. Should this be a function of declared resources? A laptop and an H100 farm should probably not have the same slot count.
- **Cache-hit pricing fraction.** What fraction of inference cost should the original producer earn on each subsequent cache retrieval? Too low (1%) and there is little incentive to contribute well-cached answers. Too high (50%) and consumers route around the cache to fresh inference. Probably 5–15% but field data needed.
- **Cross-role rate update cadence.** Every quarter is arbitrary. Could be monthly or yearly.
- **What happens at the community-commercial boundary when caching kicks in.** If a community-layer peer's answer is retrieved by a commercial query, how does the community producer get paid? In NCS? In fiat? Mediated by the requesting peer? This is RFC-0006 territory but the question lives here too.

---

## A reference implementation you could write in an afternoon

The economic layer of ANTS is, deliberately, small enough that a competent engineer can implement a minimal working version in one sitting. To prove it, here is a compact sketch in Python.

```python
import time
from collections import defaultdict, deque

class PeerRecord:
    def __init__(self, peer_id):
        self.peer_id = peer_id
        self.served_to = 0.0
        self.served_by = 0.0
        self.recent_received = deque(maxlen=120)  # (timestamp, ncs)
        self.recent_given    = deque(maxlen=120)
        self.quality = 1.0
        self.choked = True

    def credit_received(self, ncs):
        self.served_by += ncs
        self.recent_received.append((time.time(), ncs))

    def credit_given(self, ncs):
        self.served_to += ncs
        self.recent_given.append((time.time(), ncs))

    def generosity_score(self, window_seconds=1200):
        cutoff = time.time() - window_seconds
        return sum(n for t, n in self.recent_received if t >= cutoff)


class CommunityEconomy:
    def __init__(self, slots=8, optimistic_ratio=0.125):
        self.peers = defaultdict(lambda: PeerRecord(None))
        self.slots = slots
        self.optimistic_ratio = optimistic_ratio

    def unchoke_round(self):
        opt = max(1, int(self.slots * self.optimistic_ratio))
        earned = self.slots - opt

        ranked = sorted(
            self.peers.values(),
            key=lambda p: p.generosity_score() * p.quality,
            reverse=True,
        )
        winners = set(ranked[:earned])

        cold = [p for p in self.peers.values()
                if not p.recent_given
                and p not in winners]
        winners.update(cold[:opt])

        for p in self.peers.values():
            p.choked = p not in winners
```

About 40 lines of Python for the heart of the community-layer economy. Everything else — peer discovery, transport, hardware attestation, service definitions, cache integration — is plumbing around this core. We mention this not because the code is good (it is not production-grade) but because we want the *idea* to be small enough that an interested engineer can sit with it for a Saturday and form their own opinion.

---

## What comes next

This RFC defines the community layer's economy. Six other RFCs are needed before the protocol is implementable end-to-end:

- **[RFC-0002 — Semantic Cache](./RFC-0002-semantic-cache.md).** The distributed memory layer this economy depends on. Foundational.
- **[RFC-0003 — Verification](./RFC-0003-verification.md).** How peers prove the work they claim to have done.
- **[RFC-0004 — Reputation Chain with PoUH Consensus](./RFC-0004-reputation-pouh.md).** Global misbehaviour ledger; novel consensus mechanism.
- **[RFC-0005 — Identity](./RFC-0005-identity.md).** Hardware attestation; one-CPU-one-peer.
- **[RFC-0006 — Payment Terms](./RFC-0006-payment-terms.md).** How peers declare their economic terms; how multiple economies coexist.
- **RFC-0007 — Post-v1.0 Governance.** How decisions about all of the above are made once v1.0 is reached.

These will arrive in roughly that order. Each is a draft, like this one — written to be argued with, not to be obeyed.

---

## How to contribute to this document

Open an issue on `Ants-Community/ants` with the tag `rfc-0001`. The current draft will move forward in three ways: typo-and-clarity PRs (merged quickly); design-question issues (discussed in the issue, then either resolved with a PR or escalated to the Matrix room); and proposed *amendments*, which are themselves drafted as patches against this file with rationale. We keep a `CHANGELOG.md` at the bottom of `spec/` so the history of decisions is preserved.

The document is CC0. Quote it. Disagree with it in public. Fork it.

---

*Drafted by the ANTS founding contributors, May 2026.
The colony has no center. This document does not either.*

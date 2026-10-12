## Findings

**What makes the MLP arms work on this task is spread in the *training starts*. Spread elsewhere in the demos (via jitter) doesn't help, and neither does spreading the starts only at evaluation. Once the training starts are spread, the score regularizer is what makes the policies reliable.**

Success by training set (groups) and arm (bars): on each set's own evaluation, then with every policy rolled out from the single nominal start and from the same 1024 draws of C's start distribution (`scripts/skate/plot_p2p_bars.py`).

Four datasets, 4 MLP arms × 10 seeds each, n = 1024 (C is the paper's runs):

| trained on | MLP | MLP+score | MLP+phase | MLP+phase+score |
|---|---:|---:|---:|---:|
| **fixed**: one start, one goal pose, 2 distinct demos | 0% | 0% | 0% | 0% |
| **+via**: one start, via jitter (0.16 m spread mid-path) | 0% | 0% | 0% | 0% |
| **+start**: C's start noise, no via jitter | 60% | **97%** | 79% | **95%** |
| **C** (paper): start noise + via jitter + goal noise | 63% | 87% | 72% | 89% |

Success on each set's own evaluation (its held-out starts: one pose for fixed / +via, C's start distribution for +start / C).

* **Via jitter alone doesn't help.** It even makes the collisions more uniform: 80–100% of seeds drive straight into the block, against 20–80% with no jitter. The likely reason: the jitter spreads the demos from the fork outward while both modes still occupy the same few centimetres around the centre line. So the fork is densely covered by left and right demos at the same states, and every arm learns their average, which is straight ahead. More coverage there is more of the averaging trap, not an escape from it.
* **Start noise alone recovers everything, and more.** With 8 cm / 0.05 rad start noise and no via jitter, the arms match or beat C. The score arms reach 95–97% success against 86–89% on C, from the same start distribution. In the noisy-start data the side is independent of the start (corr 0.03). Start noise works because it spreads the demos where the paths part, so that the side becomes predictable from the position there. The phase-conditioned optimum then sends every start left of centre to the left (Prop. 1; App. B.2).
* **Here the score regularizer does its job.** On +start the score arms cut MMD ×10³ from 206 (MLP+phase) to 16–20 (p ≈ 0.005). They raise success from 79% to 95–97%, with an 18-point gain (p = 0.07) and a much smaller seed spread (± 2–6 vs ± 27). They drive into the block 0% of the time, against 9% for MLP+phase and 17% for the MLP. Phase on its own isn't significant: MLP 60% vs MLP+phase 79%, p = 0.14. On the one-start sets the score target is ~0 away from the fork (`fig_p2p_targets`), and the score arms are as bad as the rest.

**Cross-evaluation (training starts vs evaluation starts):** every policy rolled out from the same single nominal start, and from the same 1024 draws of C's start distribution.

| trained on | single start → goal (seeds of 10): MLP / +score / +phase / +ph.+score | C start distribution → success % |
|---|---|---|
| fixed | 0 / 0 / 0 / 0 | 0 / 0 / 0 / 1 |
| +via | 0 / 0 / 0 / 0 | 1 / 0 / 1 / 0 |
| +start | 3 / **9** / 6 / 6 | 59 / 96 / 79 / 95 |
| C | 0 / **8** / 1 / **8** | 61 / 86 / 72 / 88 |

* **Policies trained from one start fail from a spread of starts too** (0–1%). Spreading the starts only at evaluation does nothing; the damage is in what they learned.
* **Policies trained on spread starts mostly succeed from the one exact start.** That is the hardest start there is, the dead centre line where the two modes are exactly symmetric. The score arms trained on C reach the goal from it in 8 of 10 seeds; MLP+phase in 1 and the MLP in 0. Trained on +start, MLP+score manages 9, MLP+phase 6 and MLP+phase+score 6. The offset sweep (`fig_p2p_cross_offset`) shows the C-trained MLP+phase needs the start ≥ 4 cm off centre before most seeds commit to a side, while the C-trained score arms already succeed in 50–80% of seeds at 0–1 cm. The score term makes the policy break the symmetry decisively instead of averaging the modes.
* So the answer to "is stochasticity essential": yes for these deterministic policies, and specifically **variation in the training starts**. It is what lets a deterministic policy be multimodal (different starts → different modes). Once it is there, the score regularizer turns it into a sharp, symmetric split with few collisions, even from the dead-centre start.

**Mechanism, from the restart test** (policy placed exactly on a held-out demo mid-trajectory): on every point-to-point set, all arms follow the demo 100% of the time from phase ≥ 0.3, so none of this is about tracking a known path. The failures are all at the fork (phase ≈ 0.15), where on the one-start sets the two modes share the same states.

Caveats: 10 seeds per cell. The one-start sets' success is 0/1 per seed, so their means are counts. MMD compares each policy with its own set's held-out demos, so it isn't comparable *across* sets (the +start demos are much narrower than C's, which ends at scattered goals with free headings). Success is comparable: every set uses a 0.6 m goal radius around (5.0, 7.0).

**Follow-up: how much start noise is needed** -- mixtures of 5, 10 and 15% noisy-start demos with the fixed ones (`scripts/skate/start_mix_eval.py`; its report is not included here). 5% already takes every arm from 0 to mostly succeeding; 10% saturates at 96–99% success from C's starts.

**Follow-up: the start spread itself** -- the spread varied with everything else fixed, random against sorted sides, on C's held-out starts and a lateral line sweep (`scripts/skate/start_spread_eval.py`; its report is not included here). 4 cm suffices for every arm (99–100%); more hurts the unregularized arms; random sides beat sorted; the rollouts sort the starts in reverse, because the demos counter-steer.

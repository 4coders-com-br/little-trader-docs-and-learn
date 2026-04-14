# Little Trader — Expanded Curriculum

> This is the deep curriculum for the Learn surface.
>
> It is meant to be interactive, cumulative, and tightly connected to the product.

---

## Curriculum Philosophy

Little Trader should teach in four parallel directions:

1. **Programming literacy**
2. **Financial literacy**
3. **Operational literacy**
4. **Strategic literacy**

The goal is not only to help a user click buttons.
The goal is to help a user become:

- a stronger thinker
- a stronger builder
- a stronger trader
- a stronger operator

---

## Phase 0 — Orientation

### Goal
Understand what Little Trader is and how to learn inside it.

### Topics
- what the platform is
- dashboard vs REPL vs Learn vs Mel
- idea of inspectability
- what Learn is for
- why this is not only a trading app

### Lesson examples
- Welcome to Little Trader
- The 4 surfaces of the system
- Why inspection matters more than guessing
- How Mel helps without replacing judgment

### UI integration
- highlight dashboard
- highlight REPL
- highlight Learn
- point to Mel panel

---

## Phase 1 — Clojure Foundations

### Goal
Learn how to think in data and functions.

### Topics
- lists, vectors, maps, sets
- immutable updates
- functions
- sequence operations
- destructuring
- namespaces
- pure functions
- side effects

### Lesson examples
- Reading a map
- Transforming a sequence
- Modeling market data as Clojure data
- Functions as reusable thought tools

### Interactive snippet examples
```clojure
{:bar/open 100
 :bar/high 105
 :bar/low 99
 :bar/close 103}
```

```clojure
(map :brick/direction
     [{:brick/direction 1}
      {:brick/direction -1}
      {:brick/direction 1}])
```

### REPL exercises
- inspect data shapes
- transform sample bars
- compute simple summaries

---

## Phase 2 — REPL First Thinking

### Goal
Make inspection a habit.

### Topics
- REPL as microscope
- runtime inspection
- tiny probes
- safe evaluation
- debugging with curiosity

### Lesson examples
- Ask before acting
- Small probes, not giant guesses
- Use count, keys, select-keys, get-in first
- Validate understanding before changing code

### Interactive snippet examples
```clojure
(count (:bars @com.little-trader.ui.state/app-state))
```

```clojure
(select-keys @com.little-trader.ui.state/app-state [:bars :signals :open-trade])
```

### UI integration
- highlight REPL page
- highlight guided snippets
- open prefilled snippet from Learn

---

## Phase 3 — Basic Finance

### Goal
Build intuition for money, returns, and risk.

### Topics
- what an asset is
- price, return, and percentage change
- volatility
- time horizon
- compounding
- drawdown
- risk vs reward

### Lesson examples
- Why a return is not enough without risk
- The difference between a gain and a robust strategy
- Volatility as movement, not morality
- Time horizon changes behavior

### Interactive snippet examples
```clojure
(defn pct-change [a b]
  (* 100.0 (/ (- b a) a)))

(pct-change 100 110)
```

### Simulations
- compare steady returns vs volatile returns
- compare same return with different drawdowns

---

## Phase 4 — Trading Foundations

### Goal
Understand the structure of a trade.

### Topics
- entry
- exit
- stop-loss
- take-profit
- position size
- expectancy
- win rate vs risk/reward
- discipline

### Lesson examples
- A trade is a bounded thesis
- Why stop-loss is a promise, not a punishment
- Position sizing as survival logic
- Why expectancy matters more than ego

### Interactive config examples
```edn
{:entry :one-brick-reversal
 :stop-loss-percent 0.7
 :risk-per-trade 0.5}
```

### UI integration
- highlight trade panel
- highlight stop-loss controls
- compare 0.5%, 1%, 2% risk visually

---

## Phase 5 — Renko and Strategy Logic

### Goal
Understand how Little Trader sees structure.

### Topics
- bars vs bricks
- Renko noise filtering
- brick size
- one-brick / double-brick / multi-brick logic
- confirmation filters
- trend vs reversal thinking

### Lesson examples
- A brick is a compression of movement
- Brick size changes what the system sees
- A reversal is not the same as a trend
- Confirmation adds patience but costs speed

### Interactive snippets
```clojure
(map :brick/direction bricks)
```

```clojure
(take-last 2 (map :brick/direction bricks))
```

### Simulations
- compare small vs large brick size
- compare reversal sensitivity
- visualize signal delay vs noise

---

## Phase 6 — Risk Management

### Goal
Teach risk as central, not secondary.

### Topics
- capital preservation
- max risk per trade
- trailing stops
- drawdown control
- portfolio heat
- exposure discipline

### Lesson examples
- The first job is survival
- A stop defines identity under pain
- Trailing stops are dynamic promises
- Drawdown changes psychology and strategy quality

### UI integration
- highlight risk card
- pulse stop-loss field
- compare conservative vs aggressive templates

---

## Phase 7 — Options and Derivatives

### Goal
Introduce nonlinear payoff thinking.

### Topics
- calls and puts
- intrinsic vs extrinsic value
- expiration
- Greeks: delta, gamma, theta, vega
- convexity
- payoff diagrams
- spreads
- volatility as a product

### Lesson examples
- Options are structured asymmetry
- Time decay is not evil; it is a force
- Volatility changes the game before price does
- Greeks are sensitivities, not decorations

### Interactive snippets/configs
```edn
{:strategy :bull-call-spread
 :long-call {:strike 100 :premium 6}
 :short-call {:strike 110 :premium 2}}
```

### Simulations
- payoff diagrams
- theta decay examples
- volatility shock scenarios

---

## Phase 8 — Crypto and Multi-Asset Thinking

### Goal
Understand the domain beyond one chart.

### Topics
- spot vs perpetuals
- leverage
- funding
- liquidity
- exchange risk
- correlation
- diversification
- cash and hedge layers

### Lesson examples
- Spot and derivatives are different realities
- Leverage amplifies both insight and stupidity
- Correlation is hidden concentration
- Cash is a position too

### Interactive portfolio configs
```edn
{:crypto 0.35
 :usd-cash 0.25
 :equities 0.20
 :futures-hedge 0.10
 :gold 0.10}
```

### Simulations
- concentration stress
- correlation shock
- defensive rebalancing

---

## Phase 9 — LLM Collaboration

### Goal
Teach good use of Mel and AI.

### Topics
- prompting
- AI limitations
- advisory vs executable
- asking for critique
- asking for comparison
- structured questioning

### Lesson examples
- Mel is a mentor, not an oracle
- Ask for comparison, not certainty
- Use AI to widen judgment, not outsource it
- A good question is half the insight

### Exercises
- ask Mel to critique a rule set
- ask Mel for a conservative alternative
- ask Mel to explain trade risk simply

### UI integration
- mode switch: Teacher / Wizard / Strategist / Troubleshooter / Companion
- highlight relevant UI sections during answer

---

## Phase 10 — Machine Learning Fundamentals

### Goal
Demystify ML and make it practical.

### Topics
- features
- labels
- train vs test
- overfitting
- evaluation metrics
- confidence vs certainty
- regime sensitivity

### Lesson examples
- ML is pattern extraction, not prophecy
- A feature is a chosen lens on reality
- Overfitting is memorized fantasy
- Models degrade when regimes change

### Exercises
- inspect hypothetical feature sets
- compare strong vs weak labels
- analyze false confidence scenarios

---

## Phase 11 — Neural Networks and Limits

### Goal
Give serious intuition without mystification.

### Topics
- neurons, layers, weights
- forward pass
- loss
- training
- inference
- representation learning
- limitations and misuse

### Lesson examples
- Neural nets are function approximators, not magic
- A model can be impressive and still be unsafe
- More complexity does not guarantee more truth
- Interpretability matters more under risk

### Exercises
- explain a tiny network conceptually
- compare linear model vs NN intuition
- discuss when not to use NN

---

## Phase 12 — Strategy Research and Validation

### Goal
Teach research discipline.

### Topics
- hypothesis
- backtest
- scenario analysis
- robustness
- walk-forward thinking
- sensitivity analysis
- edge vs illusion

### Lesson examples
- A strategy is a hypothesis under pressure
- Backtests are stories unless validated carefully
- Sensitivity matters more than one pretty outcome
- Robustness beats brilliance

### Exercises
- compare parameter ranges
- inspect fragile vs robust strategy
- ask Mel for failure modes

---

## Phase 13 — Personal Hedge Fund Ambition

### Goal
Frame the long-term identity path.

### Topics
- operator mindset
- journaling
- decision review
- process over impulse
- building a rules culture
- capital stewardship
- ambition with discipline

### Lesson examples
- Running capital is moral weight, not just excitement
- Discipline must survive both winning and losing
- Research process is identity made visible
- Ambition needs systems, not only desire

### Companion prompts
- What kind of operator am I becoming?
- What do my rules reveal about my fears?
- How do I grow in seriousness without becoming cold?

---

## Teaching Mechanics

Each lesson should support:
- short explanation
- deep explanation
- UI highlight
- REPL snippet
- simulation or comparison
- ask Mel follow-up
- save insight / journal note

---

## Progression Model

Suggested levels:
- Beginner
- Learner
- Operator
- Builder
- Strategist
- Fund Mind

---

## Product Mapping Rule

Every lesson should answer:
- What is this concept?
- Why does it matter?
- Where do I see it in the product?
- How do I inspect it?
- How do I practice it?

---

## Final Principle

> Learn should not be separate from the product.
>
> Learn should be the path by which the user becomes capable of operating the product with judgment.

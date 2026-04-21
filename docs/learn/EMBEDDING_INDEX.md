# Learn Embedding Index

> This index is intended for later embedding inside the app.
>
> It provides a stable registry of markdown content that can be rendered in the Learn tab.

---

## Core Learn Identity

- `README.md` — what Learn is and why it exists
- `embedded-manual.md` — operator mental model
- `course-curriculum.md` — brain-layer curriculum framing
- `repl-first.md` — REPL as thinking discipline

---

## Little Trader Curriculum

- `little-trader-learning-path.md` — high-level progression
- `little-trader-expanded-curriculum.md` — deep curriculum roadmap
- `curriculum-index.md` — lesson-track navigation

### Lesson files
- `01-clojure-and-repl-foundations.md`
- `02-finance-foundations.md`
- `03-trading-foundations.md`
- `04-options-and-derivatives.md`
- `05-crypto-and-multi-asset.md`
- `06-repl-mastery.md`
- `07-llm-and-ai-collaboration.md`
- `08-machine-learning.md`
- `09-neural-networks.md`
- `10-strategy-and-hedge-fund-thinking.md`

---

## Product Integration Specs

These are docs-first implementation references that should later be rendered, linked, or used to generate app-native lessons.

- `learn-ui-lesson-system.md`
- `interactive-snippet-spec.md`
- `mel-teaching-scripts.md`

---

## Future Family Context

These are supporting vision docs outside `/learn` but relevant to Learn embedding and cross-product curriculum.

- `../docs/LEARN_TAB_PRODUCT_SPEC.md`
- `../docs/AI_MENTOR_LEARN_TAB_AND_MEL_WIZARDS.md`
- `../docs/LITTLE_FAMILY_PLATFORM_VISION_AND_LEARN_CROSS_CONCERNS.md`
- `../docs/LITTLE_BUSINESS_PRODUCT_VISION.md`
- `../docs/LITTLE_STORE_PRODUCT_VISION.md`
- `../docs/LITTLE_SYS_PLATFORM_ARCHITECTURE.md`

---

## Suggested Embedding Metadata Model

Example shape for future app registry:

```edn
[{:id :learn/welcome
  :title "Welcome to Learn"
  :path "learn/README.md"
  :track :core
  :level :beginner}
 {:id :learn/repl-foundations
  :title "Clojure and REPL Foundations"
  :path "learn/01-clojure-and-repl-foundations.md"
  :track :programming
  :level :beginner}]
```

---

## Status

This file is intentionally markdown-first so the app can later ingest it as:
- manual index
- build-time registry
- runtime lesson catalog seed

---

## Testing & Quality Assurance

These references connect Learn content to the test infrastructure:

- `../docs/TESTING_STRATEGY.md` — comprehensive test pyramid, coverage map, and conventions
- `../test/com/little_trader/browser/learn_test.clj` — browser integration tests for Learn tab
- `../test/com/little_trader/browser/smoke_test.clj` — smoke tests for SPA shell and static assets
- `../docs/SRE_OPERATIONS.md` — operational runbooks, monitoring, and incident response

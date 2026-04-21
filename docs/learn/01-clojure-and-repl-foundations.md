# 01 — Clojure and REPL Foundations

## Goal

Understand how to think in data and use the REPL as your primary interface.

---

## Lesson 1 — Data First

### Concept
Everything is data:
- maps
- vectors
- sequences

### Example
```clojure
{:price 100 :volume 200}
```

### Exercise
- add a new key
- extract value

---

## Lesson 2 — Transformations

```clojure
(map inc [1 2 3])
```

### Exercise
- transform prices

---

## Lesson 3 — REPL Loop

1. Write small expression
2. Run
3. Observe
4. Adjust

### Snippet
```clojure
(count [1 2 3 4])
```

---

## MEL Script

Teacher mode:
- explain immutability
- highlight REPL panel
- ask: "what do you expect this returns?"

---

## UI Hooks

- highlight :repl/panel
- button: Run in REPL

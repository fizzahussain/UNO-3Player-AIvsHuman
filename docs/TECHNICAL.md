# ⚙️ UNO AI — Technical Documentation

This document expands on the implementation inside `game.ipynb`, focusing on the game state, search algorithms, chance modelling, evaluation functions, and experiment flow.

---

## 🧩 1. State Representation

### Card Model

Each card is represented by the `Card` class with two values:

- `colour`
- `value`

A card is legal when either its colour or value matches the current top card.

The class also defines equality, hashing, string representation, `Skip` detection, and custom deep-copy behaviour.

### Game State

The search functions receive a dictionary containing:

```text
p1          Player 1 hand
p2          Player 2 hand
p3          Player 3 hand
top_card    Current card on the discard pile
deck        Remaining draw deck
discards    Previously played top cards
```

`apply_move()` deep-copies this state before applying a candidate action, allowing search branches to be evaluated independently without mutating the live game.

---

## 🂠 2. Deck & Legal Moves

### Deck Construction

`deck_generator()` creates:

- number cards `0–9` for each of four colours
- one `Skip` card for each colour

This produces a 44-card simplified UNO deck before shuffling.

### Legal Move Generation

`get_valid_moves()` scans a player's hand and returns cards whose colour or value matches the current top card.

For a hand of size `h`, this operation is `O(h)`.

---

## 🛡️ 3. Defensive Minimax

### Node Roles

`minimax()` evaluates the current player relative to a target AI player:

- target AI turn → `MAX`
- other player turn → `MIN`
- depth limit → heuristic leaf
- empty hand → terminal win/loss

### Terminal Scores

A win for the target AI receives a score around `+1000`, while another player's win receives approximately `-1000`. Remaining search depth is included so earlier wins and later losses are preferred.

### Defensive Evaluation

```text
50 - 6(CAI) + 2(Copp) + 4(S)
```

This puts the strongest pressure on reducing the AI's own hand and gives additional value to retained Skip cards.

### Alpha-Beta Pruning

The function carries `alpha` and `beta` values through recursive calls and stops exploring when:

```text
beta <= alpha
```

Worst-case complexity remains `O(b^d)`, but good move ordering can reduce the explored tree substantially.

---

## ⚡ 4. Offensive Expectimax

`expectimax()` treats uncertainty differently from Minimax.

### Node Roles

- Player 2's turn → `MAX`
- opponent turns → average of available actions
- forced deck draw → chance node when applicable
- depth limit → offensive evaluation function

### Offensive Evaluation

```text
50 - 5(CAI) + 3(Copp) + 2(S)
```

Relative to the defensive strategy, this increases the reward for opponents carrying more cards and lowers the emphasis on preserving Skip cards.

---

## 🎲 5. Chance Nodes

`_chance_node()` is used when the Expectimax player must draw.

The deck is first grouped by `(colour, value)`. If a card type appears `k` times in a deck of size `D`, its probability is:

```text
k / D
```

Each possible draw is applied to a copied state and recursively evaluated. The chance node returns the probability-weighted sum of those outcomes.

This makes the draw model deterministic for a given state: it evaluates all represented card outcomes instead of sampling one random future card during search.

---

## ⏭️ 6. Skip Propagation

`apply_move()` identifies which player should be skipped after a Skip card is played. The search functions carry skipped players through a set and consume that flag when the affected turn is reached.

The live `Game_UNO` loop mirrors the same behaviour during actual gameplay.

---

## 🌳 7. Search Tree Representation

The notebook defines a `TreeNode` type to store:

- node label
- node type
- heuristic/search score
- child nodes

Several helper functions calculate subtree widths and print trees in console-friendly layouts.

The output differentiates nodes such as:

```text
MAX
MIN
OPP
ACTION
CHANCE
LEAF
TERMINAL
PRUNED
```

`tree_demo()` creates a fixed example state and runs both Minimax and Expectimax at depth 3 so their decisions and search structures can be inspected side by side.

---

## 🎮 8. Game Controller

`Game_UNO` manages the actual match.

### Initialization

A shuffled deck is created, five cards are dealt to each player, and one top card is selected. If the initial top card is a Skip, the constructor attempts to replace it with a normal starting card.

### Turn Selection

The controller delegates decisions as follows:

```text
P1 -> Minimax / defensive
P2 -> Expectimax / offensive
P3 -> Human in manual mode, Minimax in simulation mode
```

### End Condition

After each move, all three hands are checked. The first player whose hand reaches zero cards becomes the winner.

A maximum-turn limit prevents simulations from running indefinitely.

---

## 🧪 9. Algorithm Comparison

`compare_algorithms()` runs repeated AI-only games while suppressing the verbose per-turn output.

For each game it records whether the winner was:

- Player 1 — Minimax
- Player 2 — Expectimax
- Player 3 — Minimax
- no winner before the turn limit

The helper supports both deterministic and randomized seeds.

The current menu invokes:

```python
compare_algorithms(n_games=50, randomize=False)
```

This is useful for comparing behaviour under repeatable deck sequences before experimenting with fully randomized seeds.

---

## 📊 10. Complexity Notes

Let:

- `b` = average action branching factor
- `d` = search depth
- `c` = distinct draw outcomes
- `h` = hand size
- `D` = remaining deck size

| Operation | Complexity |
|---|---:|
| Legal move generation | `O(h)` |
| Minimax worst case | `O(b^d)` |
| Alpha-beta ideal case | `O(b^(d/2))` |
| Build card-frequency map | `O(D)` |
| Chance-node expansion | up to `c` recursive outcomes |
| State copying | proportional to the copied game state |

The notebook uses depth 3 for normal AI decisions, which keeps the search practical for the reduced card set while still exposing multi-turn consequences.

---

## 🧭 11. Design Trade-offs

The implementation is intentionally educational and inspectable.

Some choices favour clarity over raw performance:

- each search action deep-copies the entire state
- tree nodes can be constructed for visualization
- the reduced deck limits the number of card types
- opponent actions in Expectimax are averaged uniformly
- search depth is fixed rather than dynamically adjusted

These choices make the differences between Minimax and Expectimax easier to observe directly in notebook output.

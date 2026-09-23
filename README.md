# RuleChip

**Swap the circuit, change the game.**

Game rules today are code you have to trust. RuleChip turns them into **circuits you can prove**: on-chain NAND circuits taped out with [TapeOut](https://tapeout.net) on **X Layer**. Anyone can call them for free, nobody can change them, and each one is small enough to be verified exhaustively.

> 🎮 **Live demo:** `https://lichao01111-dot.github.io/rulechip/`  
> 🎬 **Video:** `<(https://x.com/lichao0111/status/2102792970403450901/video/1)>`  
> 🐦 **X post:** `<[link](https://x.com/lichao0111/status/2102792970403450901)>`

Built for the **TapeOut Genesis Transistor Hackathon** (IGNIX × X Layer × TapeOut).

---

## On-chain deployment

| | |
|---|---|
| Network | X Layer mainnet (chain ID 196) |
| Processor | [`0xDa659F36172E644424D4C009BdCEC23818F0a2F9`](https://www.oklink.com/xlayer/address/0xDa659F36172E644424D4C009BdCEC23818F0a2F9) |
| Transistors (ERC-1155) | `0xF8192924a95754EfCA2Ba850DD79c2CF488A0D3c` |
| Call interface | `eval(uint256 circuit, bytes input) view returns (bytes)`: free, no wallet needed |

| Circuit ID | Rule | `eval` index | Size | Behaviour |
|---|---|---|---|---|
| `1.2.224` | **WrapWall** | 1 | 72 NAND · 374 transistors | Crossing a wall brings you out on the opposite side |
| `2.2.224` | **DeadWall** | 2 | 70 NAND · 364 transistors | Hitting a wall ends the game |
| `3.2.224` | **RuleMux** | 3 | 22 NAND | Picks between two rule outputs; used to build **HybridWall** |

**HybridWall = RuleMux(WrapWall, DeadWall)**: the top and bottom walls kill, the left and right walls wrap. This third rule is built entirely by wiring together two circuits that were already on-chain. No rule logic was taped out a second time.

---

## How it works

```
Player input ─► Game engine ─► encode (x, y, dir) → 8 bits
                                        │
                                        ▼
                         Processor.eval(circuit, input)   ← TapeOut circuit on X Layer
                                        │
                                        ▼
                 decode 7 bits → nextX, nextY, dead ─► render
```

The Snake engine handles rendering, input, the snake's body and food. **It contains no wall logic at all.** Every wall decision comes from the circuit that is plugged into the rule slot, so swapping the cartridge means changing one circuit index, and the engine code stays exactly the same.

### Rule interface: `TGRS-WALL-1`

The 8×8 board uses 3-bit coordinates. Bits are packed **little-endian**: bit 0 is the first entry in the BLIF `.inputs` / `.outputs` list.

| | bit 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|---|---|---|---|---|---|---|---|---|
| **Input** (1 byte) | x2 | x1 | x0 | y2 | y1 | y0 | d1 | d0 |
| **Output** (1 byte) | nx2 | nx1 | nx0 | ny2 | ny1 | ny0 | dead | – |

Directions are encoded as `0 = UP (y−1) · 1 = RIGHT (x+1) · 2 = DOWN (y+1) · 3 = LEFT (x−1)`.

Both wall rules share one datapath. First, pick x or y depending on the direction. Next, run it through a 3-bit ±1 unit. Finally, take the carry-out as the out-of-bounds signal. The two rules differ only in one wire: **DeadWall** connects the carry to `dead`, and **WrapWall** drops it.

### RuleMux (composition)

RuleMux has 15 inputs packed into 2 bytes: `a[7]` (bits 0–6) is the output of rule A, `b[7]` (bits 7–13) is the output of rule B, and `h` (bit 14) is the select signal. It returns `h ? a : b`. For HybridWall, `h = d0`, which is 1 for left/right moves, so horizontal moves use WrapWall and vertical moves use DeadWall.

```
hybrid(in) = eval(3, eval(1, in) | eval(2, in) << 7 | d0 << 14)
```

---

## Verification

A rule is a finite truth table, so RuleChip doesn't audit rules, it **proves** them by calling every input on-chain:

| Check | Result |
|---|---|
| WrapWall `1.2.224`: all 256 inputs vs. reference rule | **256 / 256** |
| DeadWall `2.2.224`: all 256 inputs vs. reference rule | **256 / 256** (32 inputs give `dead=1`) |
| RuleMux `3.2.224`: all 2¹⁵ inputs (local netlist simulation) | **32768 / 32768** |
| HybridWall end-to-end on-chain (768 chained `eval` calls) | **256 / 256** (16 inputs give `dead=1`) |
| On-chain results vs. local simulation of the taped-out BLIF | identical |

You can run the same check yourself: the demo's **Verify on-chain** button fires all the calls live and colours a 16×16 truth-table grid (rows = high nibble, columns = low nibble).

Check a single input from the command line:

```bash
# x=7, y=0, RIGHT on DeadWall → 0x40 (dead = 1)
cast call 0xDa659F36172E644424D4C009BdCEC23818F0a2F9 \
  "eval(uint256,bytes)" 2 0x87 --rpc-url https://rpc.xlayer.tech

# same input on WrapWall → 0x00 (wraps to x=0, alive)
cast call 0xDa659F36172E644424D4C009BdCEC23818F0a2F9 \
  "eval(uint256,bytes)" 1 0x87 --rpc-url https://rpc.xlayer.tech
```

---

## Optimistic play, on-chain audit

Calling the chain on every game tick adds 150–600 ms of latency, and the composite rule needs two network round trips per step. RuleChip splits the work in two:

1. **Play:** when a cartridge is inserted, all of its outputs are read from the chain into a lookup table, so the game runs with no latency.
2. **Audit:** every step is re-checked in the background against a fresh `eval` on the chain. Confirmed steps appear as "✓ step N confirmed on-chain". If a result doesn't match, the game stops immediately and shows both values.

It's the same idea as an optimistic rollup: execute first, prove right after.

### Performance notes

- One `eval` on a ~70-NAND circuit costs about **900k gas**; RuleMux costs about 340k. The public RPC caps a single `eth_call` at 50M gas.
- Calls are batched through **Multicall3** (`0xcA11bde05977b3631167028862bE2a173976CA11`), 40 wall evals per batch, with batches sent in parallel. Loading HybridWall went from about 18 s to about 4–7 s.
- Wall tables that are already loaded are reused, so switching to HybridWall afterwards only needs the RuleMux round (about 1.5 s). Verification never uses this cache and always calls the chain again.
- If the RPC can't be reached, the page falls back to a clearly labelled snapshot of earlier on-chain results.

---

## Run locally

The demo is a single static file with no build step and no dependencies.

```bash
git clone https://github.com/<your-username>/rulechip
open rulechip/index.html        # or serve the folder with any static server
```

Before you rely on the page, check that **Rule source** shows the green `on-chain eval × 256 → lookup table ✓` label. If it shows the amber **offline snapshot** label, the page couldn't reach the chain.

---

## Repository layout

```
index.html                         Demo: Snake, rule cartridges, on-chain verification, audit
circuits/
  wall_wrap.blif                   WrapWall  – flat NAND netlist (taped out as 1.2.224)
  wall_dead.blif                   DeadWall  – flat NAND netlist (taped out as 2.2.224)
  rule_mux.blif                    RuleMux   – flat NAND netlist (taped out as 3.2.224)
tools/
  gen_wallrule.py                  Generates + exhaustively verifies the WallRule netlists
  gen_rulemux.py                   Generates + verifies RuleMux and the HybridWall pipeline
rulechip_onchain_verification.json Processor, bit layout, and full on-chain output tables
```

The circuits are generated as **flat** NAND netlists, because X Layer doesn't yet accept circuits that reference sub-circuits. Each netlist is imported into the TapeOut canvas as `.blif` and taped out to the Processor above.

---

## Why TapeOut

| | Rule as code | Rule as a TapeOut circuit |
|---|---|---|
| Can the operator change it? | Yes, silently | No, a taped-out circuit is permanent |
| Can anyone verify it? | Only by auditing source code | Yes, by calling every input on-chain |
| Can others reuse it? | Copy and paste | Call the same Circuit ID, for free |
| Can it be composed? | Ad-hoc | Wire circuits together like chips (RuleMux) |
| Who owns it? | Whoever runs the server | The circuit has a unique ID; the author earns through transistor mints |

---

## Roadmap

- **Now (on-chain):** Composable Snake. Three circuits live on X Layer, all exhaustively verified.
- **Next:** *TapeArcade*. Several games share rule circuits: collision, scoring, turns, randomness.
- **Standard:** *TGRS*. A shared input/output interface for rule slots, playing the role ERC standards play for tokens.
- **Economy:** *Rule Market*. Community authors publish rule chips and earn from calls and transistor mints.
- **Frontier:** *Provable Arenas*. Prize competitions and AI-agent matches with neutral rules and replay-verifiable results.

---

## 中文简介

RuleChip 把游戏规则做成 TapeOut 链上电路。贪吃蛇引擎里没有任何撞墙逻辑，每一步都把 (x, y, 方向) 打包成 8 bit，交给 X Layer 上的电路计算：换一块电路，就换一套规则；把两块已上链的电路用 RuleMux 连起来，就得到第三种规则（HybridWall：上下撞死、左右穿墙），不需要重新流片任何规则逻辑。每条规则的全部 256 种输入都在链上穷举验证。游戏按链上加载的查找表零延迟运行，每一步在后台由链上 `eval` 审计（先走后验）。

---

## License

MIT

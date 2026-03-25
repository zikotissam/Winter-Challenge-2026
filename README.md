# Winter Challenge 2026 (SnakeBird) — 3rd Place

## The Game

SnakeBird is a 2-player game on a gravity-based grid. Each player controls multiple "birds" (snake-like creatures) that move on platforms made of walls and apples. Birds grow by eating apples, fall when unsupported, and lose their head (or die entirely if too small) when crashing into walls or other birds. The grid is horizontally symmetric, and the game ends when all apples are gone or one side is eliminated. Highest total body mass wins.

The gravity mechanic is what makes this game unique and brutal: every single move triggers a cascade of fall resolution — individual falls, then intercoiled group falls — that must all be simulated correctly. Get the simulation wrong and your search explores fantasy states.

## What Made This Contest Different For Me

In past contests, my workflow was always the same loop: spend a long time building an idea, implement it, test it, realize it's not enough, add another algorithm layer, spend forever debugging the simulation... rinse and repeat. It's slow and exhausting.

This time I changed the process completely. **I used AI as a co-pilot throughout the entire contest**, and it transformed how fast I could iterate.

Instead of spending hours debugging a single simulation edge case, I could describe what the game rules should be and get a working draft in minutes. Instead of going back and forth wondering "should I try MCTS or minimax here?", I could discuss tradeoffs and get confirmation — or better alternatives — right away. The AI didn't play the game for me. I guided every architectural decision. But it removed the friction that normally burns half my contest time.

More importantly, **discussions with the AI helped me validate or discard ideas early**. In past contests I'd commit to an approach, spend hours implementing it, then discover it was a dead end. This time, I could sketch an idea, talk through the edge cases, and either commit with confidence or pivot before writing a single line — a huge time saver.

## The Bot: DUCT with Prior-Guided Search

The final bot is a **MCTS/DUCT** (Decoupled UCT) searcher. Each bird gets its own UCT tree, and moves are selected simultaneously — the "decoupled" part — which handles the multi-agent nature of the game (up to 4 birds per side) without the combinatorial explosion of joint-action trees.

### Simulation Engine

The core of everything. A full SnakeBird simulation covering movement, apple eating, beheading, and the tricky gravity cascade (individual falls + intercoiled group falls). I used version-stamped grids instead of clearing the grid with `memset` every simulation step — a simple trick that saved a massive amount of time inside the tight MCTS loop. State copying targets only alive snake bodies with `memcpy`, not the entire structure.

### Gravity-Aware BFS

On turn 1, I precompute a distance map from every apple to every reachable cell using Dijkstra with asymmetric movement costs: moving UP (against gravity) costs more than horizontal or downward moves. This gives the evaluation a realistic sense of "how hard is it to actually reach this apple?" rather than naive Manhattan distance. The map feeds both the evaluation function and the move priors.

### Evaluation Function

The eval combines multiple weighted signals:

- **Body length** — the primary objective
- **Safety** — is there something solid below the head? Falling is dangerous
- **Fragility** — birds with 3 or fewer parts are one beheading away from death
- **Apple distance** — closest-apple distance using the gravity-aware BFS
- **Apple density** — bonus for being near clusters of apples
- **Apple control** — who's closer to each remaining apple, me or the opponent?
- **Endgame** — when no apples remain, score difference dominates with tiebreak on total losses

All weights are exposed as compile-time defines, which feeds directly into automated tuning.

### Prior-Guided UCB

Instead of vanilla UCB1, each candidate move gets a prior probability based on: wall avoidance, ground support, whether the move gets closer to apples, body collision risk, and enemy head proximity. This steers the search toward promising moves early — critical when you only have 45ms per turn. Moves that lead to guaranteed death at the root are pruned entirely.

### Time Management

First turn gets ~980ms (BFS precomputation + deep search). Subsequent turns get 45ms. Search depth adapts dynamically: fewer alive birds means deeper search, going up to depth 20 in late-game situations.

## The Tooling: Where AI Really Shined

This is where the biggest productivity gain happened. With AI assistance, I built a complete local training infrastructure in parallel with the bot itself:

### Local C++ Referee

I rebuilt the CodinGame referee entirely in C++, porting the grid generation from the original Java source. It communicates with bots via pipes, exactly like the CG server. Having a fast local referee was the foundation — running thousands of games against the CG server would take forever, but locally it takes seconds.

### Optuna TPE Parameter Tuner

The bot exposes 10 tunable parameters (search depth, UCB exploration constant, all evaluation weights, normalization divisor, climb cost) as compile-time macros. I built an automated tuner using **Optuna with TPE** (Tree-structured Parzen Estimator) that:

- Compiles the bot with candidate parameters
- Runs batches of games against multiple opponents
- Reports win-rate as the optimization objective
- Uses Bayesian optimization to converge efficiently

I also built a **genetic algorithm** variant with population-based search, elite preservation, and multi-opponent weighted evaluation for broader exploration.

### Iterative Self-Play Trainer

A master script runs self-play cycles: each cycle promotes the winner as the new baseline, then the next generation must beat it to be promoted. This prevents regression — the bot can only get stronger. Training stops when no candidate can beat the current champion (Nash equilibrium).

### Visual Testing Tools

I built Tkinter-based GUIs for both match testing and training management. The arena GUI runs bot-vs-bot matches with live stats: win-rate, average game length, score distributions. The trainer GUI controls the entire optimization process visually — select which parameters to tune, set ranges, add/remove opponents, monitor convergence in real-time. Being able to *see* the training converge rather than staring at log files made a huge difference in staying productive and making fast decisions.

## What Worked

- **AI-accelerated development**: The single biggest win. What used to take days of debugging (gravity simulation, intercoiled falls, edge cases) was done in hours. Bouncing algorithmic ideas off the AI saved me from going down dead ends I would have otherwise committed to.
- **Automated parameter tuning**: The Optuna/GA pipeline found parameter combinations I never would have hand-picked. The final tuned weights are non-obvious — no human would arrive at them through intuition alone.
- **Self-play training loop**: Once the bot could beat all test opponents, self-play kept pushing strength higher without needing new sparring partners.
- **Fast local referee**: Without this, the entire tuning pipeline wouldn't exist. Speed of iteration is everything in a contest.

## What Didn't Work

- **Server timeouts**: The CG game server gave me timeouts that never happened locally. The bot was tuned to use 45ms per turn — well within limits — but server load caused sporadic timeouts that cost games. Finishing 3rd with avoidable timeouts is frustrating. The bot was stronger than its final rank.
- **Depth tuning rabbit hole**: I spent time testing deep search (depth 14+), but extensive tuning confirmed that **depth 10-11 is the sweet spot**. Going deeper sounds better in theory, but the simulation is expensive due to gravity cascades, and good priors with shallow search consistently beat deep search with noisy rollouts.

## Key Takeaway

The meta-lesson from this contest isn't about algorithms. The bot uses a well-known technique (MCTS/DUCT), and the evaluation function is straightforward. What got me to 3rd place was **the speed at which I could iterate**: build the simulation, test it, tune parameters, fix issues, and repeat — faster than ever before.

Using AI as a coding co-pilot + building a full local training stack with visualization + automated parameter optimization — that combination let me cover in one contest what used to take three.

If there's one piece of advice: **invest in your tooling**. A bot with an okay evaluation but perfectly tuned weights will beat a bot with a brilliant evaluation and hand-picked weights every time.

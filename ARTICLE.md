# Which Task Actually Started It

A pager fires at 2am. By the time anyone opens the dashboard, nine tasks in the nightly pipeline are already red. Ingestion, two cleaning jobs, a join, both warehouse loads, both aggregates, and the report at the end of the chain. Nothing in the UI says which of the nine is the actual problem and which eight are just casualties.

This is the ordinary shape of a data pipeline incident. A pipeline is a dependency graph, and a graph does not fail politely, one task at a time, waiting for someone to diagnose the first before the second breaks. It fails the way dominoes fail: something upstream goes wrong, and everything reachable from it goes down within minutes. The on call engineer's real job in that moment is not fixing anything yet. It is figuring out, from a wall of red, which single task is the domino that got tipped over first.

The usual answer to that question is a habit, not a method: open the tasks in the order they ran, assume the earliest failure is the cause, and start reading logs from there. It works often enough that nobody questions it. It also fails silently and expensively when two unrelated things break close together, or when the graph has enough branching that "earliest" stops being a reliable proxy for "root cause."

## Treating it as inference instead of a hunch

The idea behind this project, `causal-rca`, is to replace that habit with a model that can actually be interrogated: given the set of tasks that failed, what is the probability distribution over which one started it?

That requires a generative story for how failure spreads. The one used here is a noisy-OR causal network, a classical tool from Bayesian network theory (Pearl, 1988) that fits pipeline failure almost too well. Every edge in the dependency graph carries a transmission probability: if the parent task fails, the child fails with that probability, independently of whatever else is going on. Every task also carries a small background leak probability, the chance it fails on its own, for reasons the graph does not capture. A task fails if any of its parents transmitted failure to it, or if it leaked on its own. Fix one task as "the root," let it fail with certainty, and the rest of the graph's failure pattern falls out as a consequence.

That generative model does two jobs at once. Run forward as a Monte Carlo simulator, it manufactures realistic synthetic incidents to test against. Run as inference in the other direction, given an observed set of failed tasks, it can be used to ask which starting point most plausibly produced that pattern.

## Where the shortcut breaks, and admitting it

Exact inference under this model is expensive once the graph has more than one path between two tasks, so the practical move is a closed-form shortcut: propagate marginal failure probabilities forward through the graph in one pass, treating each task's parents as independent. This is exact on a polytree, a graph where no two parents of the same task share an ancestor. It is an approximation everywhere else, because two parents that trace back to a common ancestor are not actually independent: if that ancestor failed, both of them become more likely to fail together, and the shortcut does not know that.

Rather than wave that caveat away, the project measures it. The synthetic pipeline has exactly one such convergence point, a report task fed by two aggregates that both ultimately depend on the same warehouse load. Comparing the shortcut's prediction against fifty thousand Monte Carlo trials at that node gives a mean absolute error of 0.028, against 0.001 everywhere else in the graph. Twenty six times larger, concentrated at precisely the kind of node, an aggregation or a report near the end of a pipeline, that an operator is most likely to actually care about. That is the kind of bug that never throws an exception. The number it returns is still a valid probability between zero and one. It is just wrong, quietly, exactly where it matters most.

## Letting the evidence argue back

Statistics alone will never see a deploy log. So the ranking produced by the causal model is paired with a small agent that reads the operational trail an engineer would actually pull up: a deploy event, an error message, a metric snapshot, for each of the top candidates. It scores how incriminating that evidence looks, and its opinion is combined with the statistical posterior through log-linear fusion, the standard way to merge two independently calibrated probabilistic judgments without letting either one silently dominate.

For reproducibility, the agent that reads this evidence is a deterministic, dependency-free policy by default, not a live model call, so the entire evaluation runs offline at zero cost and produces the same numbers on every run. The same interface accepts a real Claude model as a drop-in replacement for interactive use.

## The number that mattered was the uncomfortable one

Run against a thousand simulated incidents, the honest result is not a clean win. The pure statistical ranking, using an uninformative prior over which task is the root, actually loses to the naive "earliest failure" heuristic overall: 86.7 percent versus 88.2 percent top-one accuracy. A uniform prior throws away exactly the information the naive heuristic gets for free, that failure tends to move forward through a graph, not backward.

The result worth publishing was not the aggregate, it was the split. On the roughly six percent of incidents where the naive heuristic and the statistical ranking actually disagree with each other, which is the only place a smarter method has anything to prove, the pure statistical model does noticeably worse than the naive guess, 35 percent against 60 percent. Fusing in the evidence agent recovers about half of that gap, to 50 percent, but does not close it. That is a real, measured limit, not a caveat added for humility. It says plainly that the fix is not more agent evidence, it is a better prior, one that already encodes the graph's topology instead of treating every candidate as equally likely.

Reporting a negative result honestly is more useful than a headline number that only survives if nobody checks it. A platform team deciding whether to invest in something like this needs to know exactly where it helps and where it does not yet, not a single accuracy figure with the hard cases averaged away.

## Try it

The project is small on purpose: a synthetic pipeline, a noisy-OR simulator, a closed-form ranking, an evidence agent, and 26 automated tests including a brute-force exact-inference check on a hand-built graph. No paid API access is required to reproduce any of the numbers above.

Repository: https://github.com/julianodutraa/causal-rca

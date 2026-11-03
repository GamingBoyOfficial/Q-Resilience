# Q-Resilience

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/GamingBoyOfficial/Q-Resilience/blob/main/Q-Resilience.ipynb)

Estimating how likely a network is to get cut off, using quantum amplitude estimation (IQAE) in Qiskit.

You give it a network (links with failure probabilities, plus a source and a target) and it estimates the probability that the source can no longer reach the target. Every answer is checked against exact enumeration and a plain Monte Carlo baseline. We also look at what the quantum circuit costs and how noise affects it.

Built for the UWQC x IBM Qiskit Fall Fest 2026 hackathon. Everything runs on a simulator, and we don't claim a speedup on real hardware today.

## Try it (no install)

1. Click the **Open in Colab** badge above.
2. In Colab choose **Runtime > Run all**. It takes about 3 minutes.

The first cell installs the packages. If Colab asks you to restart the runtime after that, restart it and run all again.

## Use your own network

Open the config cell (Cell 1). There are two ways to define the network:

- **Edit it by hand:** change `GRAPH_EDGES` and `FAILURE_PROBS`, and leave `NETWORK_CSV = None`.
- **Load a CSV:** set `NETWORK_CSV` to a file path (or to the CSV text itself) and set `CSV_SOURCE` / `CSV_TARGET`.

The CSV has one row per link:

```
u,v,p_fail
s,a,0.10
s,b,0.15
a,b,0.05
a,t,0.12
b,t,0.08
```

`p_fail` must be strictly between 0 and 1. A bad CSV (missing header, duplicate link, source and target not connected, and so on) stops with an error message that says what is wrong. A ready example is in `example_bridge_network.csv`. To use it in Colab, upload it in the left file panel and set `NETWORK_CSV = "example_bridge_network.csv"`.

## How it works

1. Each link gets one qubit, and a rotation encodes its failure probability, so one register holds every failure pattern at once.
2. An oracle flips an extra qubit for the patterns where source and target are disconnected. We built two:
   - **v0** lists every disconnecting pattern and adds one multi-controlled gate per pattern. It is simple, but building it already needs a classical enumeration of all 2^m patterns.
   - **v1** works out "can reach" step by step through the graph, with no enumeration. It uses more qubits but far fewer gates.
3. Iterative quantum amplitude estimation (IQAE) estimates the probability of that qubit being 1.
4. We compare against exact enumeration and Monte Carlo.

## Results

All numbers come from a simulator. IQAE was run with target precision 0.02 and 95% confidence; Monte Carlo used 20,000 samples.

We ran two example networks. The first is defined by hand in the config cell. The second is loaded from `example_bridge_network.csv`.

| Method | Network 1: two parallel paths (4 links, edited in the config cell) | Network 2: bridge network (5 links, loaded from CSV) |
|---|---|---|
| Exact enumeration | 0.0658 | 0.0255 |
| Monte Carlo (20,000 samples) | 0.0625 | 0.0255 |
| IQAE with the v0 oracle | 0.0654 | 0.0255 |
| IQAE with the v1 oracle | 0.0654 | 0.0258 |

The Monte Carlo value for network 2 happens to land on the exact value; with 20,000 samples that is a lucky draw. Both oracles also match exact enumeration on 10 test networks (worst error about 7e-15).

The four figures below were produced on network 1.

### Error vs. number of calls

In state-preparation calls, IQAE's certified precision improves about 1/N (fitted slope -1.00), while Monte Carlo improves about 1/sqrt(N). This counts calls, not run time, since one quantum call is a whole circuit.

![Error vs calls](figures/qresilience_error_vs_oracle.png)

### Noise

With noise on every gate, the circuit breaks down quickly. With the k = 0 circuit the estimate is already about 4 times too high at a 0.1% single-qubit error rate. The k = 1 line looks accurate, but that is a coincidence of the fully mixed state, not robustness.

![Noise vs depth](figures/qresilience_noise_vs_depth.png)

### Oracle cost, v0 vs. v1

On a 3x3 grid, v1 needs about 190 Toffoli-equivalents against about 62,000 for v0, but 117 qubits against 13. These are compile-time counts, not hardware measurements.

![v0 vs v1](figures/qresilience_v0_vs_v1.png)

### Resource growth of the v0 circuit

Qubits grow roughly linearly with the number of links, while gates and depth grow very fast. A multi-controlled gate counts as one gate here, so the real cost is higher.

![Resource scaling](figures/qresilience_resource_scaling.png)

## Limitations

- Links fail independently. Real failures are often correlated.
- Results come from a simulator. The v0 and v1 oracles were checked against exact enumeration up to about 22 qubits. Larger v1 circuits are counts only and were not simulated.
- The advantage is only in the number of state-preparation calls for a given certified precision. With noise it disappears.
- v0 needs a classical 2^m enumeration to build. v1 avoids that but uses more qubits.

## Related work

We don't claim a new algorithm. Quantum amplitude estimation for network and power-system reliability has been studied before. Our contribution is an open, cross-checked Qiskit implementation with a resource and noise analysis.

- Grinko, Gacon, Zoufal, Woerner, "Iterative Quantum Amplitude Estimation", npj Quantum Information 7, 52 (2021), arXiv:1912.05559
- Quantum network-reliability algorithm with a reachability operator, arXiv:2203.10201
- Nikmehr and Zhang, "Quantum Distribution System Reliability Assessment", IEEE PES General Meeting, 2022

## Files

| File | What it is |
|---|---|
| `Q-Resilience.ipynb` | The notebook with all code, results and plots |
| `example_bridge_network.csv` | Example CSV input |
| `figures/` | The four figures from the notebook |
| `requirements.txt` | Packages for running it locally |

Tested with Python 3.13, Qiskit 2.6, qiskit-aer 0.18 and qiskit-algorithms 0.4.

## Team

YOUR TEAM NAME: YOUR NAME and TEAMMATE NAME

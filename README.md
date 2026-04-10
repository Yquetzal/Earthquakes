# Methodology for Building the Earthquake Cluster Interaction Graph

## 1. Objective
The goal is to detect non-random temporal relationships between spatial earthquake communities.  
Instead of analyzing individual events directly, the method first groups events into spatial clusters, then tests whether activity in one cluster is followed by activity in another cluster more often than expected by chance.

## 2. Spatial Clustering
Earthquakes are clustered using **HDBSCAN** on geographic coordinates (`latitude`, `longitude`).

HDBSCAN labels outliers as `-1` (noise). These noise points are excluded from inter-cluster graph inference.

### Configuration
- `min_cluster_size = 20`
- `min_samples = 5`
- clusters with fewer than `12` events are removed

## 3. Directed Co-occurrence Counting
For each ordered pair of clusters `(i, j)`, we compute how often an event in cluster `i` is followed by at least one event in cluster `j` within a time window `Δt`.

Default window:
- `Δt = 7` days (`7 * 24 * 3600` seconds)

The counting is **directed** and **greedy**:
- each target event in cluster `j` can be matched at most once
- this avoids inflating counts from repeated overlaps in dense periods

This produces an observed directed co-occurrence matrix.

## 4. Null Model
To test significance, we generate surrogate catalogs by randomly permuting event times while keeping the spatial structure and event set fixed.

Default settings:
- `n_shuffle = 100`
- optional parallel computation (`n_jobs = 4`)

For each shuffled catalog, the same directed co-occurrence matrix is recomputed, yielding a null distribution for every pair $(i, j)$.

## 5. Empirical p-values
For each ordered pair $(i, j)$, an empirical p-value is computed as:

$$
p_{ij} = \frac{1 + \NB\{C^{\text{null}}_{ij} \ge C^{\text{obs}}_{ij}\}}{N + 1}
$$

where $N$ is the number of shuffles.

The $+1$ correction prevents zero p-values in finite Monte Carlo samples.

## 6. Graph Construction
A directed edge $i \rightarrow j$ is added when both criteria are satisfied:
- `p_value < 0.05`
- observed co-occurrence count `>= 5`

Self-loops are removed.

The resulting graph is a directed network of clusters, where edges represent temporal associations stronger than expected under randomized timing.

## 7. Interpretation
The final network should be interpreted as a **statistical interaction map** between spatial seismic communities:
- nodes: spatial clusters
- directed edges: significant temporal precedence relationships
- no edge: insufficient evidence beyond chance under the chosen null model and thresholds

# From Rosetta to AlphaFold: What Deep Learning Taught Us About Protein Folding (and Science)

## Problem Formulation

The knowledge of the 3D structure of a folded protein is valuable for understanding its function (function lies in structure, not in sequence), designing drugs (predicting the shape of a binding pocket), interpreting genetic variants (determining whether a mutation would destabilize a fold), protein engineering, and inferring protein–protein or protein–complex interactions. The protein structure prediction problem is to predict the 3D structure of a protein from its amino-acid sequence. In practice, the input is x = (x₁, x₂, ..., x_L), where x_i is one of 20 amino acid residues and L is the sequence length. In addition to the sequence, evolutionary context and, optionally, the coordinates of already-solved structures of related sequences are used as auxiliary inputs to aid prediction.

The output is a characterization of the structure in 3D space. The simplest way is to predict 3D coordinates for each atom, where the ground truth is defined up to global rotation and translation (an equivalence class). Another approach is to predict a distance matrix for every residue pair. This representation is invariant to rotation and translation, but it does not directly give a structure; a separate coordinate search must be performed to find a conformation that satisfies the distance matrix.

---

## The Evolution of Assumptions

From a biophysics perspective, the folded protein should be at a minimum-energy state. Under this assumption, the problem becomes a search over the space of all possible structures. The naive structure space without any constraints is enormous, growing exponentially with sequence length (roughly O(2^L) or more, depending on the level of discretization).

Initially, Rosetta cut a long sequence into segments of 3–10 residues and estimated the local structure of these segments before assembling them into a global prediction. This worked reasonably well for shorter proteins. However, the bottleneck lay in tertiary structure—specifically, predicting contacts between long-range residue pairs. One might ask: *Is there any rule for estimating long-range contacts? I imagine it's still the minimum-energy principle—so why doesn't it work as well as for local structure? Why isn't this a hierarchical problem that builds from local to global in search of minimal energy? What changes from local to long-range that makes the prediction need to be data-driven rather than inferred from first principles?*

The key issue is twofold. First, the number of possible long-range contact pairs grows as O(L²) — for a 300-residue protein, that's nearly 45,000 pairs to consider. Second, the energy landscape for long-range contacts is rugged and highly non-convex, with many local minima separated by high barriers. The energy differences between near-native and decoy structures are often tiny (the "energy gap problem"), making ab initio search computationally intractable. Local fragments, by contrast, can be sampled from known structures because the sequence–structure relationship for short fragments is relatively well captured by statistical potentials. But for long-range contacts, the search space explodes, and evolutionary information becomes essential.

Subsequent works began extracting remote residue-pair contact information from co-evolution matrices. The novel assumption here is: *Residue pairs that co-evolve across homologous genes are likely in physical contact in the folded protein.* Homologous protein-coding genes have different sequences, but they perform the same function. The function is preserved because the fold remains stable despite sequence changes. When one residue mutates, its remote contact residue may mutate accordingly to compensate and maintain the contact, thereby preserving the structure and function. The homologs are organized into an N × L matrix, where N is the number of homologs and L is the sequence length.

---

## Same Assumption, Modeling Matters

Models before AlphaFold used essentially the same information as AlphaFold, so why did AlphaFold achieve such a huge performance gain? I realized I've always taken for granted that deep learning models perform better than statistical or basic machine learning models without understanding where the gain comes from.

It's easy to see why a deep learning model would excel in image classification compared to traditional methods. A CNN is a more complex and flexible function approximator. Compared to, say, using manually designed filters to extract low-level visual features and then combining them with a linear layer, a CNN treats the filters themselves as learnable parameters. Moreover, the inductive bias that features from pixels to semantics lie in a hierarchy—where higher-level (more semantically rich) features are aggregates of local lower-level visual features—is both strong and accurate for image data.

Of course, AlphaFold also benefited from larger and deeper MSAs (via HHblits and JackHMMER), a larger and cleaner training set (PDB with careful filtering), iterative recycling, and distillation from predicted structures. But beyond these engineering advances, there were fundamental architectural differences that I want to focus on.

---

## What Old Models Missed

Analogous to CNNs being a better fit for images, what is AlphaFold's inductive bias for amino-acid sequences, the target structure, and their relationship? What did older models miss with their own inductive biases?

From 2D CNNs to attention: earlier deep-learning models used CNNs to slide across cropped L × L co-evolution matrices to capture relationships between residues. However, the receptive field remained local. The attention mechanism, on the other hand, allows distant residues to directly interact with each other. A CNN aggregates different input dimensions based solely on their positions in the grid, independent of their values. In contrast, attention aggregates in a value-dependent and location-agnostic way:

```
h_i^(l+1) = Σⱼ a_ij (W_v hⱼ),  a_ij = softmaxⱼ(q_i kⱼ / √d),  q_i = W_q h_i,  kⱼ = W_k hⱼ
```

For a 1D CNN:

```
h_i^(l+1) = Σ_{k=-K}^{K} W_k h_{i+k}^l
```

The weight with which hⱼ is aggregated into the next layer depends only on j-i in a 1D CNN. In attention, besides the position-independent W_v, there is an extra weight a_ij that depends on the values of both the query and the key. For example, attention can "attend strongly to j only if both i and j are cysteines"—a pattern that a fixed-kernel CNN cannot express without an exponential blow-up in the number of channels to cover every content combination separately. For a given channel c, we have:

```
h_{c,i}^(l+1) = Σ_{k=-K}^{K} W_k h_{i+k}^l
```

For i and j within the same receptive field, they contribute as W_i h_i^l + W_j h_j^l. To make the aggregation explicitly content-aware—i.e., to condition (W_i, W_j) on (h_i^l, h_j^l)—the model would need |h_i| × |h_j| channels, which is infeasible. Moreover, the interaction captured is not just between two residues but among all residues within the receptive field, and the number of such residues scales with kernel size. This limitation is especially problematic for amino-acid sequences, where long-range dependencies are critical.

Choosing how to characterize protein structure also matters. Predicting Cartesian coordinates for every atom (x_i, y_i, z_i) ignores the fixed relative positions of atoms within a residue, which must be enforced as constraints. AlphaFold instead treats each residue as a rigid body with its own local coordinate frame. To place all atoms of a residue into the global coordinate system, a rigid-body transformation (rotation and translation) is applied to the local frame. This design choice ensures that the prediction is equivariant to global rotation and translation — the model doesn't need to learn this invariance from data; it's built into the architecture. Essentially, the model predicts the translation and rotation for each residue, and the atom positions are derived deterministically from the local frame.

---

## New Physics Learned?

Another question I've pondered is: *Does the fact that AlphaFold can predict structures of unseen proteins well (i.e., generalize) indicate that the model has captured new, underlying physics of protein folding—whether or not those rules are legible to us?* If so, why were these rules unknown to biophysicists before? What makes them illegible? Are these rules deterministic in the sense of being the global minimum of the energy landscape, or are they local minima that evolution has selected? Why exactly is this scientific problem benefiting so much from deep learning?

I don't think AlphaFold has discovered new physical laws. Rather, it has learned a highly effective *surrogate* for the folding outcome, bypassing the energy landscape entirely. The model maps sequences directly to structures using statistical patterns extracted from evolutionary data — not by simulating forces or minimizing energy. Whether that surrogate implicitly encodes the global energy minimum, an evolutionarily selected local minimum, or something else entirely (like a structural "consensus" across homologs) is still an open question. What's clear is that AlphaFold's predictions are often physically plausible but can still violate basic constraints (e.g., steric clashes) — which suggests it hasn't fully captured physics, but rather a very good statistical approximation of it.

From the moment an amino-acid sequence is generated, the folding process involves hydrophobic collapse (where hydrophobic side chains coalesce), native-state search (where the chain twists and turns to find the lowest-energy state), and side-chain packing (where side chains rotate to form a compact structure after the backbone is set). At each step, atoms are guided by the force field created by all other atoms, moving in the direction of steepest free-energy descent. The force field is perfectly described by a system of differential equations:

```
m_i d²r_i/dt² = -∇ᵢ E(r₁, r₂, ..., r_N)
```

An analytical solution would give each atom's coordinates as a function of time, but this is only possible for the two-body problem (N=2). Even the three-body problem has no general analytical solution. Determining the folding process is essentially solving an N-body problem with N on the order of thousands. Since it cannot be solved analytically, one alternative is numerical integration: assume constant force over a short timestep, update positions, recompute forces, and repeat. However, a reasonable timestep is around 10⁻¹⁵ seconds, and folding takes about 10⁻³ seconds (for many small proteins; timescales can range from microseconds to seconds), requiring 10¹² steps—computationally infeasible, not to mention the accumulation of numerical error.

Both analytical solution and brute-force simulation are impractical. AlphaFold reformulates the question from *"What is the entire folding trajectory for a specific sequence?"* to *"What does the final folded structure look like?"* — a much more tractable mapping.

---

## The Prerequisite for "AI for Science"

In this sequence-to-structure task, the mapping from sequence to structure is many-to-one. The protein structure is robust: evolution can mutate one amino acid to a chemically similar one without changing the fold; a local motif like an alpha-helix or beta-sheet can arise from many different sequences; and long-range contacts can be preserved through correlated mutations.

This reminds me of learning a new piano piece—finding the common theme and variations saves time in developing muscle memory. A piece can be compressed into a main theme and its variations in different keys, tempos, or repetitions. As long as the interval pattern is preserved, we perceive it as the same theme. So in music, varying note sequences are invariant in the semantic space of the theme.

It's similar in vision: applying a simple filter changes pixel values, but the contrast between patches remains. The varying pixel values are invariant in the semantic space.

In protein folding, certain varying amino-acid sequences are invariant in terms of local structure and long-range contacts. For local relationships, the model learns, for example, that a pattern of hydrophobic and hydrophilic residues at a specific periodicity is likely to form an alpha-helix. For long-range relationships, the model must identify co-evolutionary patterns. If it sees that whenever position 50 has a large amino acid, position 180 has a small one, and vice versa, that is a statistical hint that these two residues are close in 3D, even though they are distant in sequence.

So what characteristics make a scientific problem likely to benefit from deep learning? Based on this case, I'd argue for several criteria:

1. **The input is high-dimensional but structured.** Sequences, images, and text all have this property — raw data is large, but there are strong regularities.

2. **The output is lower-dimensional or constrained.** Here, the output is a 3D structure with physical constraints (bond lengths, angles, no clashes). This makes the mapping learnable.

3. **There is abundant data.** For proteins, we have millions of sequences via genomics and tens of thousands of solved structures. For single-cell omics, we're accumulating similar-scale data.

4. **The underlying physics is too complex to simulate directly.** When first-principles simulation is computationally prohibitive, a data-driven surrogate can be a powerful alternative.

5. **There are strong statistical regularities.** Co-evolution, local motifs, and evolutionary conservation all provide signals that correlate with the target.

These criteria suggest that deep learning is most transformative when we have large datasets, a clear many-to-one mapping, and a problem where traditional simulation or analytical approaches hit a wall.
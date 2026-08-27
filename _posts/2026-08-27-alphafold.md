# From Rosetta to AlphaFold: What Deep Learning Taught Us About Protein Folding (and Science)
​
## Problem Formulation
​
The knowledge of the 3D structure of a folded protein is valuable for understanding its function (function lies in structure, not in sequence), designing drugs (predicting the shape of a binding pocket), interpreting genetic variants (determining whether a mutation would destabilize a fold), protein engineering, and inferring protein–protein or protein–complex interactions. The protein structure prediction problem is to predict the 3D structure of a protein from its amino-acid sequence. In practice, the input is $x = (x_1, x_2, ..., x_L)$, where $x_i$ is one of 20 amino acid residues and $L$ is the sequence length. In addition to the sequence, evolutionary context and, optionally, the coordinates of already-solved structures of related sequences are used as auxiliary inputs to aid prediction.
​
The output is a characterization of the structure in 3D space. The simplest way is to predict 3D coordinates for each atom, where the ground truth is defined up to global rotation and translation (an equivalence class). Another approach is to predict a distance matrix for every residue pair. This representation is invariant to rotation and translation, but it does not directly give a structure; a separate coordinate search must be performed to find a conformation that satisfies the distance matrix.
​
---
​
## The Evolution of Assumptions
​
From a biophysics perspective, the folded protein should be at a minimum-energy state. Under this assumption, the problem becomes a search over the space of all possible structures. The naive structure space without any constraints is enormous, growing exponentially with sequence length (roughly $O(2^L)$ or more, depending on the level of discretization).
​
Initially, Rosetta cut a long sequence into segments of 3–10 residues and estimated the local structure of these segments before assembling them into a global prediction. This worked reasonably well for shorter proteins. However, the bottleneck lay in tertiary structure—specifically, predicting contacts between long-range residue pairs. One might ask: *Is there any rule for estimating long-range contacts? I imagine it's still the minimum-energy principle—so why doesn't it work as well as for local structure? Why isn't this a hierarchical problem that builds from local to global in search of minimal energy? What changes from local to long-range that makes the prediction need to be data-driven rather than inferred from first principles?*
​
The key issue is twofold. First, the number of possible long-range contact pairs grows as $O(L^2)$ — for a 300-residue protein, that's nearly 45,000 pairs to consider. Second, the energy landscape for long-range contacts is rugged and highly non-convex, with many local minima separated by high barriers. The energy differences between near-native and decoy structures are often tiny (the "energy gap problem"), making ab initio search computationally intractable. Local fragments, by contrast, can be sampled from known structures because the sequence–structure relationship for short fragments is relatively well captured by statistical potentials. But for long-range contacts, the search space explodes, and evolutionary information becomes essential.
​
Subsequent works began extracting remote residue-pair contact information from co-evolution matrices. The novel assumption here is: *Residue pairs that co-evolve across homologous genes are likely in physical contact in the folded protein.* Homologous protein-coding genes have different sequences, but they perform the same function. The function is preserved because the fold remains stable despite sequence changes. When one residue mutates, its remote contact residue may mutate accordingly to compensate and maintain the contact, thereby preserving the structure and function. The homologs are organized into an $N \times L$ matrix, where $N$ is the number of homologs and $L$ is the sequence length.
​
---
​
## Same Assumption, Modeling Matters
​
Models before AlphaFold used essentially the same information as AlphaFold, so why did AlphaFold achieve such a huge performance gain? I realized I've always taken for granted that deep learning models perform better than statistical or basic machine learning models without understanding where the gain comes from.
​
It's easy to see why a deep learning model would excel in image classification compared to traditional methods. A CNN is a more complex and flexible function approximator. Compared to, say, using manually designed filters to extract low-level visual features and then combining them with a linear layer, a CNN treats the filters themselves as learnable parameters. Moreover, the inductive bias that features from pixels to semantics lie in a hierarchy—where higher-level (more semantically rich) features are aggregates of local lower-level visual features—is both strong and accurate for image data.
​
Of course, AlphaFold also benefited from larger and deeper MSAs (via HHblits and JackHMMER), a larger and cleaner training set (PDB with careful filtering), iterative recycling, and distillation from predicted structures. But beyond these engineering advances, there were fundamental architectural differences that I want to focus on.
​
---
​
## What Old Models Missed
​
Analogous to CNNs being a better fit for images, what is AlphaFold's inductive bias for amino-acid sequences, the target structure, and their relationship? What did older models miss with their own inductive biases?
​
From 2D CNNs to attention: earlier deep-learning models used CNNs to slide across cropped $L \times L$ co-evolution matrices to capture relationships between residues. However, the receptive field remained local. The attention mechanism, on the other hand, allows distant residues to directly interact with each other. A CNN aggregates different input dimensions based solely on their positions in the grid, independent of their values. In contrast, attention aggregates in a value-dependent and location-agnostic way:
​
$$
h_i^{l+1} = \sum_j a_{ij}(W_v h_j), \quad a_{ij} = \text{softmax}_j\left(\frac{q_i k_j}{\sqrt{d}}\right), \quad q_i = W_q h_i, \quad k_j = W_k h_j.
$$
​
For a 1D CNN:
​
$$
h_i^{l+1} = \sum_{k=-K}^{K} W_k h_{i+k}^l.
$$
​
The weight with which $h_j$ is aggregated into the next layer depends only on $j-i$ in a 1D CNN. In attention, besides the position-independent $W_v$, there is an extra weight $a_{ij}$ that depends on the values of both the query and the key. For example, attention can "attend strongly to $j$ only if both $i$ and $j$ are cysteines"—a pattern that a fixed-kernel CNN cannot express without an exponential blow-up in the number of channels to cover every content combination separately. For a given channel $c$, we have:
​
$$
h_{c,i}^{l+1} = \sum_{k=-K}^{K} W_k h_{i+k}^l.
$$
​
For $i$ and $j$ within the same receptive field, they contribute as $W_i h_i^
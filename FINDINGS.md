# VQC Experiments — Final Findings

## Notebooks
- `iris_xor_qvc.ipynb` — main Iris binary classifier
- `experiments/vqc_xor.ipynb` — XOR problem
- `experiments/vqc_moons.ipynb` — Make Moons with data re-uploading

---

## Iris classifier

**Architecture:** 2-qubit Variational Quantum Classifier.  
Encoder: `ry(pi * xA, q0)`, `ry(pi * xB, q1)`.  
Ansatz: `N_LAYERS` repetitions of a trainable layer containing `ry` rotations on both qubits followed by a `cx` entangling gate, plus a final classifier head.  
Measurement: Pauli-Z expectation value on qubit 0.  
Labels: `CLASS_A -> +1`, `CLASS_B -> -1`.  
Optimizer: COBYLA.

**Configurable knobs:**
- `CLASS_A/B`: selected Iris class pair
- `FEATURE_A/B`: selected feature pair
- `N_LAYERS`: number of variational layers

---

## Iris findings

The Iris experiments show that VQC performance is dominated by the quality of the selected features.

- **Setosa vs any other class** reaches `1.0` accuracy. This is expected because Setosa is a clearly separated outlier cluster in the Iris dataset. This result does not indicate quantum advantage, since classical models also solve this case easily.

- **Versicolor vs Virginica using sepal features `(0, 1)`** reaches only around `0.6` accuracy, even when increasing `N_LAYERS`. The sepal feature space is too overlapping for this small 2-qubit model to separate reliably.

- **Versicolor vs Virginica using petal features `(2, 3)`** reaches around `0.95` accuracy with `N_LAYERS = 1`. Petal length and petal width contain much more useful class information, so even a shallow VQC is sufficient.

- Increasing the number of layers does not fix poor feature selection. More circuit depth cannot compensate for input features that do not separate the classes well.

- On the informative petal features, the learned decision boundary is approximately linear. This explains why one variational layer is already enough.

**Classical comparisons on Iris:**  
Both an MLP (`hidden_layer_sizes=(2,)`, `tanh`) and an SVM with RBF kernel were run on the same train/test split and the same selected features. On the separable petal features, all three models reach similar test accuracy, around `0.95`. On the overlapping sepal features, the classical models also struggle, confirming that the performance ceiling is mainly caused by data geometry rather than by the VQC alone.

The 3-panel decision-surface plot shows that the VQC boundary is qualitatively similar to the SVM boundary, which suggests that in this small setting the VQC behaves like a trainable nonlinear feature-map classifier rather than something fundamentally different from classical kernel methods.

**Conclusion for Iris:**  
The VQC behaves like a small trainable classifier whose success depends mostly on the geometry of the selected feature space. The experiment demonstrates quantum data encoding, variational training, and measurement-based classification, but it does not show quantum advantage.

---

## XOR experiment

**Architecture:** same 2-qubit VQC structure as in the Iris experiment.  
Dataset: four XOR points.  
Training setup: full dataset used for training because the dataset has only four points.

---

## XOR findings

The XOR experiment shows the importance of entanglement and feature interaction.

- With `N_LAYERS = 1` and a `cx` gate, the VQC solves XOR perfectly.

- The learned decision surface is nonlinear, matching the structure required by XOR. A purely linear classifier cannot solve XOR.

- Removing the `cx` gate makes the learned boundary a straight/simple boundary. This was verified experimentally. Without entanglement, the model cannot create the interaction between the two input features that XOR requires.

- Increasing `N_LAYERS` without `cx` still does not solve the interaction problem. Repeated single-qubit rotations remain separable across the two qubits, so the model behaves like independent one-qubit transformations rather than a true two-feature interaction model.

**More precise interpretation:**  
The `cx` gate does not make quantum mechanics nonlinear. Quantum gates are linear operations on the quantum state. However, the combination of classical input encoding, entanglement, and measurement produces a nonlinear function of the original classical input features. This allows the VQC to represent the XOR decision boundary.

**Comparison with a classical MLP:**  
A small `MLPClassifier` with 2 hidden neurons and `tanh` activation was run on the same XOR data. With `random_state=42`, the MLP failed to solve XOR, converging to a poor local solution and producing a roughly linear boundary that misclassified two of the four points.

This is not a clean architectural comparison. The MLP has more parameters than the VQC, but it is trained with gradient-based optimization, while the VQC uses COBYLA, a derivative-free optimizer. The result illustrates a practical difference in optimization behavior for this run, not a fundamental quantum advantage.

**Comparison with SVM using RBF kernel:**  
The SVM with an RBF kernel is a stronger classical comparison. Both the VQC and the RBF SVM can be interpreted as models that map inputs into a richer feature space before classification. The VQC does this through quantum encoding and entanglement, while the SVM does it through the RBF kernel function.

On XOR, the SVM solves the problem perfectly, as expected. The decision surfaces are qualitatively similar: both produce smooth, curved boundaries that separate the XOR corners.

**Conclusion for XOR:**  
The XOR result demonstrates that entanglement can create useful feature interactions in a VQC. However, this is not evidence of quantum advantage. The circuit has only two qubits, is classically simulable, and behaves like a very small nonlinear parametric model.

---

## Make Moons experiment

**Architecture:** 2-qubit VQC with data re-uploading.  
Dataset: `make_moons` with 200 samples and `noise=0.15`.  
Split: 75/25 train/test split.  
Preprocessing: features normalized using train-set min/max.  
Layer structure: each layer encodes `x` via `Ry(pi*x0)` and `Ry(pi*x1)`, then applies trainable `Ry(theta)` gates on both qubits, followed by `cx`. A final classifier head completes the circuit.  
Configuration: `N_LAYERS = 3`, giving 8 total parameters.  
Optimizer: COBYLA with `maxiter=1000`.

---

## Make Moons findings

The moons experiment is the most important experiment for understanding VQC expressivity.

**Single-encoding limitation:**  
The original architecture encoded the input once at the beginning and then stacked variational layers. This version saturated around `0.80` test accuracy, even when increasing `N_LAYERS` from 2 to 10.

This suggests that, for this architecture, simply adding more trainable layers after a single input encoding did not create a sufficiently flexible decision boundary for the curved moons dataset. The issue was not only optimization; it was also a limitation of how the input information was injected into the circuit.

**Data re-uploading improves expressivity:**  
With data re-uploading, the input is re-encoded at every layer. This means each trainable block receives the input again after the quantum state has already been transformed by previous gates.

The architecture becomes:

`Encode(x) -> Trainable block -> Encode(x) -> Trainable block -> Encode(x) -> Trainable block -> Measurement`

This allows the final expectation value to depend on the input through richer combinations of sine, cosine, and interaction terms. As a result, the decision boundary becomes more flexible and can better match the curved geometry of the moons dataset.

With `N_LAYERS = 3` and data re-uploading, the VQC reaches test accuracy comparable to the SVM and MLP baselines.

**Key insight:**  
For this 2-qubit architecture, data re-uploading made depth useful. Without re-uploading, additional trainable layers gave little benefit. With re-uploading, each layer injected the input again, allowing the model to build a more expressive nonlinear decision function.

**Comparison with classical models:**  
The MLP (`hidden_layer_sizes=(2,)`) and SVM with RBF kernel both solve moons comfortably and serve as classical baselines. The 3-panel decision-surface plot shows that all three models produce smooth curved boundaries separating the two crescents.

The VQC surface after re-uploading is qualitatively similar to the SVM surface, again suggesting that the VQC behaves like a learned nonlinear feature-map classifier in this small simulated setting.

**Conclusion for Make Moons:**  
The moons experiment reveals the most actionable design principle of the project: if a small VQC is not expressive enough, simply stacking trainable gates may not help. Re-uploading the input between trainable layers can significantly increase expressivity and allow the circuit to learn curved decision boundaries.

---

## Overall conclusions

These experiments support four main conclusions:

1. **Feature selection matters more than circuit depth.**  
   On Iris, good features allow a shallow VQC to perform well, while bad features remain difficult even with more layers.

2. **Entanglement is useful when the task requires feature interaction.**  
   XOR cannot be solved by a straight-line boundary. Adding `cx` allows the VQC to create feature interactions and produce a nonlinear decision surface.

3. **Data re-uploading can make circuit depth useful.**  
   On Make Moons, a single input encoding followed by many trainable layers saturated in performance. Re-encoding the input at every layer allowed the 2-qubit VQC to learn a more flexible curved boundary.

4. **No quantum advantage is demonstrated.**  
   The experiments are small, classically simulable, and use datasets that classical models can solve. The value of the project is educational and analytical: it shows how encoding, ansatz design, entanglement, re-uploading, measurement, and optimization affect VQC behavior.

---

## Final takeaway

A VQC should not be treated as a magic classifier. Its performance depends on matching the circuit design to the structure of the data.

In these experiments:

- Iris showed that feature selection dominates performance.
- XOR showed that entanglement is needed for feature interaction.
- Make Moons showed that data re-uploading can greatly improve nonlinear expressivity.
- Classical baselines showed that these results do not imply quantum advantage.

The main lesson is that VQC design is about controlling how classical data enters, mixes, and is measured inside the quantum circuit.

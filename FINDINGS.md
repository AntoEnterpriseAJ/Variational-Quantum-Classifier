# VQC Experiments — Final Findings

## Notebooks
- `iris_xor_qvc.ipynb` — main Iris binary classifier
- `experiments/vqc_xor.ipynb` — XOR problem

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

The Iris experiments show that the VQC performance is dominated by the quality of the selected features.

- **Setosa vs any other class** reaches `1.0` accuracy. This is expected because Setosa is a clearly separated outlier cluster in the Iris dataset. This result does not indicate quantum advantage, since classical models also solve this case easily.

- **Versicolor vs Virginica using sepal features `(0, 1)`** reaches only around `0.6` accuracy, even when increasing `N_LAYERS`. The sepal feature space is too overlapping for this small 2-qubit model to separate reliably.

- **Versicolor vs Virginica using petal features `(2, 3)`** reaches around `0.95` accuracy with `N_LAYERS = 1`. Petal length and petal width contain much more useful class information, so even a shallow VQC is sufficient.

- Increasing the number of layers does not fix poor feature selection. More circuit depth cannot compensate for input features that do not separate the classes well.

- On the informative petal features, the learned decision boundary is approximately linear. This explains why one variational layer is already enough.

**Classical comparisons on Iris:**  
Both an MLP (`hidden_layer_sizes=(2,)`, `tanh`) and an SVM (RBF kernel) were run on the same train/test split and the same two selected features. On the separable petal features, all three models reach similar test accuracy (~0.95). On the overlapping sepal features, MLP and SVM also struggle, confirming that the accuracy ceiling is a data geometry problem, not a model capacity problem. The 3-panel decision surface plot shows that the VQC boundary is qualitatively similar to the SVM boundary, again consistent with the VQC behaving like a trainable kernel method.

**Conclusion for Iris:**  
The VQC behaves like a small trainable classifier whose success depends mostly on the geometry of the selected feature space. The experiment demonstrates quantum data encoding, variational training, and measurement-based classification, but it does not show quantum advantage.

---

## XOR experiment

**Architecture:** same 2-qubit VQC structure as in the Iris experiment.  
Dataset: four XOR points.  
Training setup: full dataset used for training because the dataset has only four points.

---

## XOR findings

The XOR experiment shows the importance of entanglement / feature interaction.

- With `N_LAYERS = 1` and a `cx` gate, the VQC solves XOR perfectly.

- The learned decision surface is nonlinear, matching the structure required by XOR. A purely linear classifier cannot solve XOR.

- Removing the `cx` gate makes the learned boundary a straight line. This was verified experimentally. Without entanglement, the model cannot create the interaction between the two input features that XOR requires.

- Increasing `N_LAYERS` without `cx` still does not solve the interaction problem. Repeated single-qubit rotations remain separable across the two qubits, so the model behaves like a product of independent one-qubit transformations.

**More precise interpretation:**  
The `cx` gate does not make quantum mechanics "nonlinear" by itself. Quantum gates are still linear operations on the quantum state. However, the combination of input encoding, entanglement, and measurement creates a nonlinear function of the original classical input features. This allows the VQC to represent the XOR decision boundary.

**Comparison with a classical MLP:**  
A small `MLPClassifier` with 2 hidden neurons and `tanh` activation was run on the same XOR data. With `random_state=42`, the MLP failed to solve XOR — it converged to a local minimum and produced a roughly linear boundary that misclassified two of the four points. The VQC solved XOR correctly in the same run. This is not a clean architectural comparison: the MLP has more parameters (11 vs 4) but uses gradient descent, which gets trapped on a near-flat loss landscape for small datasets. The VQC uses COBYLA, a derivative-free optimizer that is less susceptible to gradient vanishing. The result illustrates a practical difference in optimization behavior, not a fundamental quantum advantage.

**Comparison with SVM (RBF kernel):**  
The SVM with an RBF kernel is a structurally more equivalent comparison to the VQC. Both models implicitly map inputs to a higher-dimensional feature space before classifying: the VQC does this through quantum encoding and entanglement, the SVM does it via the RBF kernel function. On the small XOR dataset the SVM solves the problem perfectly, as expected. The decision surfaces look qualitatively similar — a smooth, curved boundary that separates the four XOR corners — which underlines that the VQC is behaving like a trainable kernel method, not doing something fundamentally different.

**Conclusion for XOR:**  
The XOR result demonstrates that entanglement can create useful feature interactions in a VQC. However, this is not evidence of quantum advantage. The circuit has only two qubits, is classically simulable, and behaves like a very small nonlinear parametric model.

---

## Overall conclusions

These experiments support three main conclusions:

1. **Feature selection matters more than circuit depth.**  
   On Iris, good features allow a shallow VQC to perform well, while bad features remain difficult even with more layers.

2. **Entanglement is useful when the task requires feature interaction.**  
   XOR cannot be solved by a straight-line boundary. Adding `cx` allows the VQC to produce a nonlinear decision surface.

3. **No quantum advantage is demonstrated.**  
   The experiments are small, classically simulable, and use datasets that classical models can solve. The value of the project is educational and analytical: it shows how encoding, ansatz design, entanglement, measurement, and optimization affect VQC behavior.

---

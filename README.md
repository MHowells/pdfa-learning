# pdfa-learning

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.20932562.svg)](https://doi.org/10.5281/zenodo.20932562)
[![Binder](https://mybinder.org/badge_logo.svg)](https://mybinder.org/v2/gh/MHowells/pdfa-learning/HEAD)

This repository contains code for running the grammatical inference algorithm 
ALERGIA for sequential pattern mining. It includes functions for constructing 
prefix-tree acceptors (PTAs) and probabilistic prefix-tree acceptors (PPTAs), 
learning deterministic finite automata (DFAs) and probabilistic deterministic 
finite automata (PDFAs), estimating pattern and sequence probabilities, and 
visualising the resulting automata. 

There are two approaches to the ALERGIA algorithm implemented in this codebase. 
The first is the `carrasco` approach, found in the original paper for the 
ALERGIA algorithm by Carrasco and Oncina (1994) [[1]](#1). The second is the 
`de_la_higuera` approach, that uses a red-blue framework to solve the algorithm, as 
outlined in de la Higuera (2010) [[2]](#2). The default method is `carrasco`, although 
the `de_la_higuera` approach is cheaper to compute.

## Contents

- [Installing Dependencies](#installing-dependencies)
- [Input Data](#input-data)
- [Running ALERGIA](#running-alergia)
  - [Choosing the Alpha Parameter](#choosing-the-alpha-parameter)
  - [Controlling Print Output](#controlling-print-output)
- [Probabilistic Deterministic Finite Automata](#probabilistic-deterministic-finite-automata-pdfa)
- [Examples and Tutorials](#examples-and-tutorials)
- [Running Tests](#running-tests)
- [Author ORCID](#author-orcid)
- [Funding](#funding)
- [References](#references)

## Installing Dependencies

The codebase has been tested with Python 3.11.1, with the requirements 
specified in `requirements.txt`.

Please note, the visualisation functions require GraphViz to be installed
on your system in addition to the Python dependencies. You can install the
software from the [GraphViz website](https://graphviz.org/download/).

To create a virtual environment:

    $ python -m venv env

To start using the new virtual environment:

    $ source env/bin/activate

To install the dependencies:

    $ python -m pip install -r requirements.txt

Alternatively, you can use conda to create a new environment with the required
dependencies by running the following command:

    $ conda env create --file binder/environment.yml

## Input data

The initial data is provided as a list of sequences. Each character in a 
sequence represents a symbol from the alphabet.

For example, given the following list of sequences:

```python
sequences = [
    "A", "A", "A", "A", "A", "A", "A", "A", 
    "AB", "AB", "AB", "AB", "AB", "AB", "AB", 
    "BA", "BA", "BA", "BA", "BA", 
    "BB", "BB", "BB", "BB", "BB", 
    "BC", "BC", 
    "B", "B", "B",
]
```

These sequences can be used to construct the following 
prefix-tree acceptor (PTA):

![example_ppta](https://github.com/MHowells/pdfa-learning/blob/main/figs/example_pta.svg)

The alphabet, states, and transition-count matrix for this PTA can be
constructed directly from the sequences:

```python
import pdfa_learning as pl

alphabet = pl.get_alphabet(sequences) 
states = pl.get_initial_states(sequences) 

transition_matrix = pl.get_transition_matrix(
    sequences, 
    alphabet, 
)
```

For this example, the alphabet (or actions) is:

```python 
["A", "B", "C"] 
```

The state list contains seven prefix states along with the artificial starting
state `"*"`:

```python
["*", 0, 1, 2, 3, 4, 5, 6]
```

The `transition_matrix` is a three-dimensional NumPy array. Its first dimension 
represents the alphabet symbol, its second dimension represents the current 
state, and its third dimension represents the destination state.

For example:

```python
transition_matrix[0, 1, 2]
```

contains the number of transitions from state `0` to state `1` using 
symbol `"A"`.

Note that the transition from the starting state `"*"` is stored in the first
alphabet layer of the matrix, here, `'A'`. This is not true and does not 
represent a genuine emitted symbol, but has no effect on the results of the 
algorithm.

The full transition-count matrix for this example is:

```python
np.array([
    [
        [0, 30, 0, 0, 0, 0, 0, 0],
        [0, 0, 15, 0, 0, 0, 0, 0],
        [0, 0, 0, 0, 0, 0, 0, 0],
        [0, 0, 0, 0, 0, 0, 0, 0],
        [0, 0, 0, 0, 0, 5, 0, 0],
        [0, 0, 0, 0, 0, 0, 0, 0],
        [0, 0, 0, 0, 0, 0, 0, 0],
        [0, 0, 0, 0, 0, 0, 0, 0],
    ],
    [
        [0, 0, 0, 0, 0, 0, 0, 0],
        [0, 0, 0, 0, 15, 0, 0, 0],
        [0, 0, 0, 7, 0, 0, 0, 0],
        [0, 0, 0, 0, 0, 0, 0, 0],
        [0, 0, 0, 0, 0, 0, 5, 0],
        [0, 0, 0, 0, 0, 0, 0, 0],
        [0, 0, 0, 0, 0, 0, 0, 0],
        [0, 0, 0, 0, 0, 0, 0, 0],
    ],
    [
        [0, 0, 0, 0, 0, 0, 0, 0],
        [0, 0, 0, 0, 0, 0, 0, 0],
        [0, 0, 0, 0, 0, 0, 0, 0],
        [0, 0, 0, 0, 0, 0, 0, 0],
        [0, 0, 0, 0, 0, 0, 0, 2],
        [0, 0, 0, 0, 0, 0, 0, 0],
        [0, 0, 0, 0, 0, 0, 0, 0],
        [0, 0, 0, 0, 0, 0, 0, 0],
    ]
])
```

Now that the transition-count matrix has been constructed, it can be 
visualised as the above PTA using the following function:

```python
pl.network_visualisation(
    transition_matrix, 
    states, alphabet, 
    filename="figs/example_pta", 
    save=True, 
    probabilities=False, 
    graph_format="svg",
)
```

Note here we have set `probabilities=False` to visualise the transition
counts rather than the transition probabilities. Setting `probabilities=True` 
would instead represent the transition probabilities, and hence a 
Probabilistic Prefix Tree Acceptor (PPTA).

## Running ALERGIA

The transition-count matrix can be passed to `alergia()` to learn a smaller 
DFA:

```python
learned_matrix, learned_states, tracking = pl.alergia(
    transition_matrix,
    states,
    alphabet,
    alpha=0.2,
    method="carrasco",
)
```

The `method` parameter can be either:

```python
method="carrasco"
```

or:

```python
method="de_la_higuera"
```

The function returns:

- `learned_matrix`, containing the transition counts of the learned automaton;
- `learned_states`, containing the remaining state identifiers;
- `tracking`, containing information about the attempted, successful, and 
failed merges.

### Choosing the alpha Parameter

The `alpha` parameter controls the tolerance used when deciding whether 
two states are statistically compatible and may therefore be merged. 

In this implementation: 
- smaller `alpha` values produce a wider compatibility bound and generally 
allow more state merges; 
- larger `alpha` values produce a narrower compatibility bound and generally 
result in fewer state merges. 

Consequently, smaller values tend to produce more compact, generalised 
automata, whereas larger values tend to preserve more of the structure 
present in the original prefix tree.

The appropriate value of alpha is application dependent and may be selected 
by comparing the resulting automata using validation data or other 
model selection criteria. Users may wish to use some of the evaluation 
functions contained in `evaluation.py` for this purpose.

### Controlling Print Output

The amount of information printed while the algorithm runs can be controlled 
using the `output_level` parameter:

```python
output_level="Suppressed"
output_level="Truncated"
output_level="Full"
```

- `"Suppressed"` prints no progress information and is the default. 
- `"Truncated"` prints the main state comparisons and merge outcomes. 
- `"Full"` additionally prints iteration numbers and details of recursive merges.

## Probability Deterministic Finite Automata (PDFA)

The learned transition-count matrix (DFA) can be converted into a probability 
transition matrix, or PDFA:

```python
probability_matrix = pl.probability_transition_matrix(
    learned_matrix,
    learned_states,
    alphabet,
)
```

The probability matrix can then be used by the pattern and sequence probability 
functions contained in `pdfa_learning.py`.

For example:

```python
pattern_probability = pl.probability_estimate_of_pattern(
    probability_matrix,
    pattern="01",
    alphabet=alphabet,
)

sequence_probability = pl.probability_estimate_of_exact_sequence(
    probability_matrix,
    sequence="01",
    alphabet=alphabet,
)
```

## Visualising the PDFA

Similar to the PTA visualisation, the learned PDFA can be visualised using
the `network_visualisation()` function. Note the final transition-count
matrix is passed to the function, but the `probabilities` parameter is set to
`True`.

```python
pl.network_visualisation(
    learned_matrix, 
    learned_states, 
    alphabet, 
    filename="figs/example_pdfa", 
    save=True, 
    probabilities=True, 
    graph_format="svg",
)
```

This produces the following PDFA:

![example_pdfa](https://github.com/MHowells/pdfa-learning/blob/main/figs/example_pdfa.svg)

## Examples and tutorials

A minimal runnable example is available in
[`examples/basic_usage.py`](examples/basic_usage.py).

The following step-by-step notebooks are also provided:
- [`01_basic_usage.ipynb`](examples/nbs/01_basic_usage.ipynb) –
  Construct a PTA and learn a PDFA using ALERGIA.
- [`02_visualising_a_pdfa.ipynb`](examples/nbs/02_visualising_a_pdfa.ipynb) –
  Visualise the original prefix tree and learned automaton.
- [`03_evaluating_a_pdfa.ipynb`](examples/nbs/03_evaluating_a_pdfa.ipynb) –
  Evaluate a learned PDFA on observed sequences.

A notebook is also provided to verify the results of the examples in the 
literature:
- [`04_verifying_literature.ipynb`](examples/nbs/04_verifying_literature.ipynb)

## Running tests

You can run the complete test suite using the following command:

```bash
$ python -m pytest
```

Run the tests with statement and branch coverage using:

```bash
python -m pytest \
    --cov=pdfa_learning \
    --cov-branch \
    --cov-report=term-missing
```

## Author ORCID

- Matthew Howells: [0000-0002-3931-7027](https://orcid.org/0000-0002-3931-7027)
- Paul Harper: [0000-0001-7894-4907](https://orcid.org/0000-0001-7894-4907)
- Daniel Gartner: [0000-0003-4361-8559](https://orcid.org/0000-0003-4361-8559)
- Geraint Palmer-Liyu: [0000-0001-7865-6964](https://orcid.org/0000-0001-7865-6964)

## Funding 

This code is funded by an Engineering and Physical Sciences Research Council 
(EPSRC) Enhanced CASE PhD Studentship with Cardiff and Vale University Health 
Board as the project partner (Project reference: 2601327, in relation to 
EP/T517951/1).

## Citation

If you use `pdfa-learning` in your research, please cite the software as:

> Howells, M., Harper, P., Gartner, D., & Palmer-Liyu, G. (2026). *pdfa_learning* (v.1.0.0). Zenodo. https://doi.org/10.5281/zenodo.20932563

Citation metadata are also available in [`CITATION.cff`](CITATION.cff).

```bibtex
@software{howells_2026_20932563,
  author       = {Howells, Matthew and
                  Harper, Paul and
                  Gartner, Daniel and
                  Palmer-Liyu, Geraint},
  title        = {pdfa\_learning},
  month        = jun,
  year         = 2026,
  publisher    = {Zenodo},
  version      = {v.1.0.0},
  doi          = {10.5281/zenodo.20932563},
  url          = {https://doi.org/10.5281/zenodo.20932563},
}
```

## References
<a id="1">[1]</a> 
Carrasco, R.C. and Oncina, J., (1994).
Learning stochastic regular grammars by means of a state merging method.
International Colloquium on Grammatical Inference, (pp. 139-152).

<a id="2">[2]</a> 
De la Higuera, C., (2010). 
Grammatical inference: learning automata and grammars. 
Cambridge University Press.
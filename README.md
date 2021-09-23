# 🔥 Pyrodigal [![Stars](https://img.shields.io/github/stars/althonos/pyrodigal.svg?style=social&maxAge=3600&label=Star)](https://github.com/althonos/pyrodigal/stargazers)

*Cython bindings and Python interface to [Prodigal](https://github.com/hyattpd/Prodigal/), an ORF
finder for genomes and metagenomes. **Now with SIMD!***

[![Actions](https://img.shields.io/github/workflow/status/althonos/pyrodigal/Test/master?logo=github&style=flat-square&maxAge=300)](https://github.com/althonos/pyrodigal/actions)
[![Coverage](https://img.shields.io/codecov/c/gh/althonos/pyrodigal?style=flat-square&maxAge=3600)](https://codecov.io/gh/althonos/pyrodigal/)
[![License](https://img.shields.io/badge/license-GPLv3-blue.svg?style=flat-square&maxAge=2678400)](https://choosealicense.com/licenses/gpl-3.0/)
[![PyPI](https://img.shields.io/pypi/v/pyrodigal.svg?style=flat-square&maxAge=3600)](https://pypi.org/project/pyrodigal)
[![Bioconda](https://img.shields.io/conda/vn/bioconda/pyrodigal?style=flat-square&maxAge=3600)](https://anaconda.org/bioconda/pyrodigal)
[![Wheel](https://img.shields.io/pypi/wheel/pyrodigal.svg?style=flat-square&maxAge=3600)](https://pypi.org/project/pyrodigal/#files)
[![Python Versions](https://img.shields.io/pypi/pyversions/pyrodigal.svg?style=flat-square&maxAge=600)](https://pypi.org/project/pyrodigal/#files)
[![Python Implementations](https://img.shields.io/pypi/implementation/pyrodigal.svg?style=flat-square&maxAge=600&label=impl)](https://pypi.org/project/pyrodigal/#files)
[![Source](https://img.shields.io/badge/source-GitHub-303030.svg?maxAge=2678400&style=flat-square)](https://github.com/althonos/pyrodigal/)
[![GitHub issues](https://img.shields.io/github/issues/althonos/pyrodigal.svg?style=flat-square&maxAge=600)](https://github.com/althonos/pyrodigal/issues)
[![Changelog](https://img.shields.io/badge/keep%20a-changelog-8A0707.svg?maxAge=2678400&style=flat-square)](https://github.com/althonos/pyrodigal.py/blob/master/CHANGELOG.md)
[![Downloads](https://img.shields.io/badge/dynamic/json?style=flat-square&color=303f9f&maxAge=86400&label=downloads&query=%24.total_downloads&url=https%3A%2F%2Fapi.pepy.tech%2Fapi%2Fprojects%2Fpyrodigal)](https://pepy.tech/project/pyrodigal)
[![DOI](https://img.shields.io/badge/doi-10.5281%2Fzenodo.4015169-purple?style=flat-square&maxAge=86400)](https://doi.org/10.5281/zenodo.4015169)


## 🗺️ Overview

Pyrodigal is a Python module that provides bindings to Prodigal using
[Cython](https://cython.org/). It directly interacts with the Prodigal
internals, which has the following advantages:

- **single dependency**: Pyrodigal is distributed as a Python package, so you
  can add it as a dependency to your project, and stop worrying about the
  Prodigal binary being present on the end-user machine.
- **no intermediate files**: Everything happens in memory, in a Python object
  you fully control, so you don't have to invoke the Prodigal CLI using a
  sub-process and temporary files. Sequences can be passed directly as
  strings or bytes, which avoids the overhead of formatting your input to
  FASTA for Prodigal.
- **lower memory usage**: Pyrodigal is slightly more conservative when it comes
  to using memory, which can help process very large sequences. It also lets
  you save some more memory when running several *meta*-mode analyses
- **better performance**: Pyrodigal uses *SIMD* instructions to compute which
  dynamic programming nodes can be ignored when scoring connections. This can
  save from a third to half the runtime depending on the sequence.

### 📋 Features

The library now features everything needed to run Prodigal in single or
metagenomic mode. It is still missing some features of the CLI:

**Roadmap**:

- ✔️ Metagenomic mode
- ✔️ Single mode
- ❌ External training file support (`-t` flag)
- ❌ Region masking (`-m` flag)

### 🐏 Memory

Pyrodigal makes two changes compared to the original Prodigal command line:

* Sequences are stored as raw bytes instead of compressed bitmaps. This means
  that the sequence itself takes 3/8th more space, but since the memory used
  for storing the sequence is often negligible compared to the memory used to
  store dynamic programming nodes, this is an acceptable trade-off for better
  performance when finding the start and stop nodes.
* Node arrays are dynamically allocated and grow exponentially instead of
  being pre-allocated with a large size. On small sequences, this leads to
  Pyrodigal using about 30% less memory.


### 🧶 Thread-safety

`pyrodigal.Pyrodigal` instances are thread-safe. In addition, the `find_genes`
method is re-entrant. This means you can train a `Pyrodigal` instance once,
and then use a pool to process sequences in parallel:
```python
p = Pyrodigal()
p.train(training_sequence)
with multiprocessing.pool.ThreadPool() as pool:
    predictions = pool.map(p.find_genes, sequences)
```

## 🔧 Installing

Pyrodigal can be installed directly from [PyPI](https://pypi.org/project/pyrodigal/),
which hosts some pre-built wheels for the x86-64 architecture (Linux/OSX/Windows)
and the Aarch64 architecture (Linux only), as well as the code required to compile
from source with Cython:
```console
$ pip install pyrodigal
```

Otherwise, Pyrodigal is also available as a [Bioconda](https://bioconda.github.io/)
package:
```console
$ conda install -c bioconda pyrodigal
```

## 💡 Example

Lets load a sequence from a
[GenBank](http://www.insdc.org/files/feature_table.html) file, use `Pyrodigal`
to find all the genes it contains, and print the proteins in two-line FASTA
format.

### 🔬 [Biopython](https://github.com/biopython/biopython)

To use `Pyrodigal` in single mode, you must explicitly call `Pyrodigal.train`
with the sequence you want to use for training before trying to find genes,
or you will get a [`RuntimeError`](https://docs.python.org/3/library/exceptions.html#RuntimeError):
```python
p = pyrodigal.Pyrodigal()
p.train(bytes(record.seq))
genes = p.find_genes(bytes(record.seq))
```

However, in `meta` mode, you can find genes directly:
```python
record = Bio.SeqIO.read("sequence.gbk", "genbank")
p = pyrodigal.Pyrodigal(meta=True)

for i, gene in enumerate(p.find_genes(bytes(record.seq))):
    print(f">{record.id}_{i+1}")
    print(record.translate())
```

*On older versions of Biopython (before 1.79) you will need to use
`record.seq.encode()` instead of `bytes(record.seq)`*.


### 🧪 [Scikit-bio](https://github.com/biocore/scikit-bio)

```python
seq = next(skbio.io.read("sequence.gbk", "genbank"))
p = pyrodigal.Pyrodigal(meta=True)

for i, gene in enumerate(p.find_genes(seq.values.view('B'))):
    print(f">{record.id}_{i+1}")
    print(record.translate())
```

*We need to use the [`view`](https://numpy.org/doc/stable/reference/generated/numpy.ndarray.view.html)
method to get the sequence viewable by Cython as an array of `unsigned char`.*


## 💭 Feedback

### ⚠️ Issue Tracker

Found a bug ? Have an enhancement request ? Head over to the [GitHub issue
tracker](https://github.com/althonos/pyrodigal/issues) if you need to report
or ask something. If you are filing in on a bug, please include as much
information as you can about the issue, and try to recreate the same bug
in a simple, easily reproducible situation.

### 🏗️ Contributing

Contributions are more than welcome! See
[`CONTRIBUTING.md`](https://github.com/althonos/pyrodigal/blob/master/CONTRIBUTING.md)
for more details.


## ⚖️ License

This library is provided under the [GNU General Public License v3.0](https://choosealicense.com/licenses/gpl-3.0/).
The Prodigal code was written by [Doug Hyatt](https://github.com/hyattpd) and is distributed under the
terms of the GPLv3 as well. See `vendor/Prodigal/LICENSE` for more information. The `cpu_features` library is
licensed under the terms of the Apache license. See `vendor/cpu_features/LICENSE` for more information.

*This project is in no way not affiliated, sponsored, or otherwise endorsed
by the [original Prodigal authors](https://github.com/hyattpd). It was developed
by [Martin Larralde](https://github.com/althonos/) during his PhD project
at the [European Molecular Biology Laboratory](https://www.embl.de/) in
the [Zeller team](https://github.com/zellerlab).*

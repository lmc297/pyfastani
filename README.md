# 🐍⏩🧬 PyFastANI [![Stars](https://img.shields.io/github/stars/althonos/pyfastani.svg?style=social&maxAge=3600&label=Star)](https://github.com/althonos/pyfastani/stargazers)

*[Cython](https://cython.org/) bindings and Python interface to [FastANI](https://github.com/ParBLiSS/FastANI/), a method for fast whole-genome similarity estimation.
**Now with multithreading!***

[![Actions](https://img.shields.io/github/actions/workflow/status/althonos/pyfastani/test.yml?branch=main&logo=github&style=flat-square&maxAge=300)](https://github.com/althonos/pyfastani/actions)
[![Coverage](https://img.shields.io/codecov/c/gh/althonos/pyfastani/branch/main.svg?style=flat-square&maxAge=3600)](https://codecov.io/gh/althonos/pyfastani/)
[![License](https://img.shields.io/badge/license-MIT-blue.svg?style=flat-square&maxAge=2678400)](https://choosealicense.com/licenses/mit/)
[![PyPI](https://img.shields.io/pypi/v/pyfastani.svg?style=flat-square&maxAge=3600)](https://pypi.org/project/pyfastani)
[![Bioconda](https://img.shields.io/conda/vn/bioconda/pyfastani?style=flat-square&maxAge=3600&logo=anaconda)](https://anaconda.org/bioconda/pyfastani)
[![AUR](https://img.shields.io/aur/version/python-pyfastani?logo=archlinux&style=flat-square&maxAge=3600)](https://aur.archlinux.org/packages/python-pyfastani)
[![Wheel](https://img.shields.io/pypi/wheel/pyfastani.svg?style=flat-square&maxAge=3600)](https://pypi.org/project/pyfastani/#files)
[![Python Versions](https://img.shields.io/pypi/pyversions/pyfastani.svg?style=flat-square&maxAge=600)](https://pypi.org/project/pyfastani/#files)
[![Python Implementations](https://img.shields.io/pypi/implementation/pyfastani.svg?style=flat-square&maxAge=600&label=impl)](https://pypi.org/project/pyfastani/#files)
[![Source](https://img.shields.io/badge/source-GitHub-303030.svg?maxAge=2678400&style=flat-square)](https://github.com/althonos/pyfastani/)
[![Mirror](https://img.shields.io/badge/mirror-EMBL-009f4d?style=flat-square&maxAge=2678400)](https://git.embl.de/larralde/pyfastani/)
[![Issues](https://img.shields.io/github/issues/althonos/pyfastani.svg?style=flat-square&maxAge=600)](https://github.com/althonos/pyfastani/issues)
[![Docs](https://img.shields.io/readthedocs/pyfastani/latest?style=flat-square&maxAge=600)](https://pyfastani.readthedocs.io)
[![Changelog](https://img.shields.io/badge/keep%20a-changelog-8A0707.svg?maxAge=2678400&style=flat-square)](https://github.com/althonos/pyfastani/blob/master/CHANGELOG.md)
[![Downloads](https://img.shields.io/pypi/dm/pyfastani?style=flat-square&color=303f9f&maxAge=86400&label=downloads)](https://pepy.tech/project/pyfastani)
[![Preprint](https://img.shields.io/badge/preprint-bioRxiv-darkblue?style=flat-square&maxAge=2678400)](https://www.biorxiv.org/content/10.1101/2025.02.13.638148v1)
<!-- [![Talk](https://img.shields.io/badge/talk-10.7490%2Ff1000research.1119176.1-f2673c?style=flat-square&maxAge=86400)](https://doi.org/10.7490/f1000research.1119176.1) -->


## 🗺️ Overview

FastANI is a method published in 2018 by [Chirag Jain](https://github.com/cjain7)
*et al.* for high-throughput computation of whole-genome
[Average Nucleotide Identity (ANI)](https://img.jgi.doe.gov/docs/ANI.pdf).
It uses [MashMap](https://github.com/marbl/MashMap) to compute orthologous mappings
without the need for expensive alignments.


`pyfastani` is a Python module, implemented using the [Cython](https://cython.org/)
language, that provides bindings to FastANI. It directly interacts with the
FastANI internals, which has the following advantages over CLI wrappers:

- **simpler compilation**: FastANI requires several additional libraries,
  which make compilation of the original binary non-trivial. In PyFastANI,
  libraries that were needed for threading or I/O are provided as stubs,
  and `Boost::math` headers are vendored so you can build the package without
  hassle. Or even better, just install from one of the provided wheels!
- **single dependency**: If your software or your analysis pipeline is
  distributed as a Python package, you can add `pyfastani` as a dependency to
  your project, and stop worrying about the FastANI binary being present on
  the end-user machine.
- **sans I/O**: Everything happens in memory, in Python objects you control,
  making it easier to pass your sequences to FastANI
  without needing to write them to a temporary file.
- **multi-threading**: Genome query resolves the fragment mapping step in
  parallel, leading to shorter querying times even with a single genome.

*This library is still a work-in-progress, and in an experimental stage,
but it should already pack enough features to be used in a standard pipeline.*


## 🔧 Installing

PyFastANI can be installed directly from [PyPI](https://pypi.org/project/pyfastani/),
which hosts some pre-built CPython wheels for x86-64 Unix platforms, as well
as the code required to compile from source with Cython:
```console
$ pip install pyfastani
```

In the event you have to compile the package from source, all the required
libraries are vendored in the source distribution, so you'll only need a
C/C++ compiler.

Otherwise, PyFastANI is also available as a [Bioconda](https://pyfastani.github.io/)
package:
```console
$ conda install -c bioconda pyfastani
```

## 💡 Example

The following snippets show how to compute the ANI between two genomes,
with the reference being a draft genome. For one-to-many or many-to-many
searches, simply add additional references with `m.add_draft` before indexing.
*Note that any name can be given to the reference sequences, this will just
affect the `name` attribute of the hits returned for a query.*

### 🔬 [Biopython](https://github.com/biopython/biopython)

Biopython does not let us access to the sequence directly, so we need to
convert it to bytes first with the `bytes` builtin function. For older
versions of Biopython (earlier than 1.79), use `record.seq.encode()`
instead of `bytes(record.seq)`.

```python
import pyfastani
import Bio.SeqIO

sketch = pyfastani.Sketch()

# add a single draft genome to the mapper, and index it
ref = list(Bio.SeqIO.parse("vendor/FastANI/data/Shigella_flexneri_2a_01.fna", "fasta"))
sketch.add_draft("S. flexneri", (bytes(record.seq) for record in ref))

# index the sketch and get a mapper
mapper = sketch.index()

# read the query and query the mapper
query = Bio.SeqIO.read("vendor/FastANI/data/Escherichia_coli_str_K12_MG1655.fna", "fasta")
hits = mapper.query_sequence(bytes(query.seq))

for hit in hits:
    print("E. coli K12 MG1655", hit.name, hit.identity, hit.matches, hit.fragments)
```

### 🧪 [Scikit-bio](https://github.com/biocore/scikit-bio)

Scikit-bio lets us access to the sequence directly as a `numpy` array, but
shows the values as byte strings by default. To make them readable as
`char` (for compatibility with the C code), they must be cast with
`seq.values.view('B')`.

```python
import pyfastani
import skbio.io

sketch = pyfastani.Sketch()

ref = list(skbio.io.read("vendor/FastANI/data/Shigella_flexneri_2a_01.fna", "fasta"))
sketch.add_draft("Shigella_flexneri_2a_01", (seq.values.view('B') for seq in ref))

mapper = sketch.index()

# read the query and query the mapper
query = next(skbio.io.read("vendor/FastANI/data/Escherichia_coli_str_K12_MG1655.fna", "fasta"))
hits = mapper.query_genome(query.values.view('B'))

for hit in hits:
    print("E. coli K12 MG1655", hit.name, hit.identity, hit.matches, hit.fragments)
```

## ⏱️ Benchmarks

In the original FastANI tool, multi-threading was only used to improve the
performance of many-to-many searches: each thread would have a chunk of the
reference genomes, and querying would be done in parallel for each reference.
However, with a small set of reference genomes, there may not be enough for
all the threads to work, so it cannot scale with a large number of threads. In
addition, this causes the same query genome to be hashed several times, which
is not optimal. In `pyfastani`, multi-threading is used to compute the hashes and mapping of query genome fragments. This allows parallelism to be useful even
when a only few reference genomes are available.

The benchmarks below show the time for querying a single genome (with
`Mapper.query_draft`) using a variable number of threads. *Benchmarks
were run on a [i7-8550U CPU](https://www.intel.fr/content/www/fr/fr/products/sku/122589/) running @1.80GHz with 4 physical / 8 logical
cores, using 50 bacterial genomes from the [proGenomes](https://progenomes.embl.de/) database.
For clarity, only 5 randomly-selected genomes are shown on the second graph. Each run was repeated 3 times.*

![Benchmarks](https://raw.githubusercontent.com/althonos/pyfastani/main/benches/mapping/v0.4.0.svg)

## 🔖 Citation

<!-- PyFastANI is scientific software; it was presented among other optimized
software at the [European Student Council Symposium (ESCS) 2022](https://www.escs2022.iscbsc.org/) during [ECCB 2022](https://eccb2022.org/). -->

If you found PyFastANI useful, please cite [our paper](https://academic.oup.com/nargab/article/7/3/lqaf095/8196481), as well as the original [FastANI paper](https://www.nature.com/articles/s41467-018-07641-9).

To cite PyFastANI:
> Martin Larralde, Georg Zeller, Laura M. Carroll. 2025. PyOrthoANI, PyFastANI, and Pyskani: a suite of Python libraries for computation of average nucleotide identity. *NAR Genomics and Bioinformatics* 7(3):lqaf095. doi: 10.1093/nargab/lqaf095.

To cite FastANI:
> Chirag Jain, Luis M Rodriguez-R, Adam M Phillippy, Konstantinos T Konstantinidis, Srinivas Aluru. 2018. High throughput ANI analysis of 90K prokaryotic genomes reveals clear species boundaries. *Nature Communications* 9(1):5114. doi: 10.1038/s41467-018-07641-9.

## 🔎 See Also

Computing ANI for metagenomic sequences? You may be interested in
[`pyskani`, a Python package for computing ANI](https://github.com/althonos/pyskani)
using the [`skani` method](https://www.biorxiv.org/content/10.1101/2023.01.18.524587v1)
developed by [Jim Shaw](https://jim-shaw-bluenote.github.io/)
and [Yun William Yu](https://github.com/yunwilliamyu).

## 💭 Feedback

### ⚠️ Issue Tracker

Found a bug ? Have an enhancement request ? Head over to the [GitHub issue
tracker](https://github.com/althonos/pyfastani/issues) if you need to report
or ask something. If you are filing in on a bug, please include as much
information as you can about the issue, and try to recreate the same bug
in a simple, easily reproducible situation.

### 🏗️ Contributing

Contributions are more than welcome! See
[`CONTRIBUTING.md`](https://github.com/althonos/pyfastani/blob/master/CONTRIBUTING.md)
for more details.


## ⚖️ License

This library is provided under the [MIT License](https://choosealicense.com/licenses/mit/).

The FastANI code was written by [Chirag Jain](https://github.com/cjain7)
and is distributed under the terms of the
[Apache License 2.0](https://choosealicense.com/licenses/apache-2.0/),
unless otherwise specified in vendored sources. See `vendor/FastANI/LICENSE`
for more information.
The `cpu_features` code was written by [Guillaume Chatelet](https://github.com/gchatelet)
and is distributed under the terms of the [Apache License 2.0](https://choosealicense.com/licenses/apache-2.0/).
See `vendor/cpu_features/LICENSE` for more information.
The `Boost::math` headers were written by [Boost Libraries](https://www.boost.org/) contributors
and is distributed under the terms of the [Boost Software License](https://choosealicense.com/licenses/bsl-1.0/).
See `vendor/boost-math/LICENSE` for more information.

*This project is in no way not affiliated, sponsored, or otherwise endorsed
by the [original FastANI authors](https://github.com/cjain7). It was developed by
[Martin Larralde](https://github.com/althonos/) during his PhD project
at the [European Molecular Biology Laboratory](https://www.embl.de/) in
the [Zeller team](https://github.com/zellerlab).*

# Snakemake for Lattice Quantum Field Theory

## Snakemake for Lattice Quantum Field Theory

This lesson introduces the Snakemake workflow system in the context of a lattice quantum field theory data
analysis task.

To quote from the [official Snakemake documentation](https://snakemake.readthedocs.io/):

> The Snakemake workflow management system is a tool to create reproducible and scalable data analyses.
> Workflows are described via a human readable, Python based language. They can be seamlessly scaled to
> server, cluster, grid and cloud environments, without the need to modify the workflow definition.
> Finally, Snakemake workflows can entail a description of required software, which will be automatically
> deployed to any execution environment.

Snakemake originated as, and remains most popular as, a tool for bioinformatics.
However,
Snakemake is a general-purpose system
and may be used for all manner of data processing tasks;
the examples we use here are tailored to the context of Lattice Quantum Field Theory,
which in some cases has different priorities in the design of analysis workflows.

Snakemake is a superset of the Python language and as such can draw on the full power of Python, but you
do not need to be a Python programmer to use it. This lesson **assumes no prior knowledge of Python** and
introduces just a few concepts as needed to construct useful workflows.

## Contributing

We welcome all contributions to improve the lesson! Maintainers will do their best to help you if you have any
questions, concerns, or experience any difficulties along the way.

We'd like to ask you to familiarize yourself with our [Contribution Guide](CONTRIBUTING.md) and have a look at
the [more detailed guidelines][lesson-example] on proper formatting, ways to render the lesson locally, and even
how to write new episodes.

Please see the current list of [issues][issues]
for ideas for contributing to this repository. For making your contribution, we use the GitHub flow, which is
nicely explained in the chapter [Contributing to a Project](https://git-scm.com/book/en/v2/GitHub-Contributing-to-a-Project) in Pro Git
by Scott Chacon.
Look for the tag ![good\_first\_issue](https://img.shields.io/badge/-good%20first%20issue-gold.svg). This indicates that the maintainers
will welcome a pull request fixing this issue.

## Maintainer(s)

Current maintainers of this lesson are

- [Ed Bennett](https://github.com/edbennett)

## Authors

A list of contributors to the lesson can be found in [AUTHORS](AUTHORS)

## Citation

To cite this lesson, please consult with [CITATION](CITATION)

## Acknowledgement

This lesson draws on material and structure from the [Introduction to Snakemake for Bioinformatics][bio].

The development of this lesson was funded by
the UKRI Science and Technology Facilities Council (STFC)
Research Software Engineer Fellowship EP/V052489/1.

[bio]: https://github.com/carpentries-incubator/snakemake-novice-bioinformatics
[issues]: https://github.com/edbennett/snakemake-novice-lattice/issues
[lesson-example]: https://carpentries.github.io/lesson-example

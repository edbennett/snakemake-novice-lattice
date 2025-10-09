---
site: sandpaper::sandpaper_site
---

Being able to reliably run a range of tools,
across multiple inputs,
and in the right order,
is a very common problem in computing,
including in scientific data analysis.
If you develop scripts for data analysis from scratch,
you will usually end up running up against these challenges.
Rather than reinventing the wheel,
we can lean on existing tools to do this for us,
letting us focus on the aspects unique to our own work.
Such tools are called _workflow managers_._

In this lesson,
we introduce [Snakemake][snakemake],
a workflow manager originally developed for bioinformatics applications,
but that maps well onto the needs of
data analysis in lattice quantum field theory.

By defining _rules_,
each of which specify
how to translate one or more input files into one or more output files,
we can build up a workflow that takes our raw data as input,
and produces plots, tables, and other definitions
that we can include in our publications.

### Acknowledgments

The development of this lesson was supported by
the STFC Research Software Engineer Fellowship
EP/V052489/1.
The data presented were generated using
the [Supercomputing Wales][scw] SUNBIRD system.
Supercomputing Wales is supported by
the [European Regional Development Fund][erdf]
via [Welsh Government][welshgov]

[erdf]: https://commission.europa.eu/funding-tenders/find-funding/eu-funding-programmes/european-regional-development-fund-erdf_en
[scw]: https://www.supercomputing.wales
[snakemake]: https://snakemake.github.io/
[welshgov]: https://www.gov.wales

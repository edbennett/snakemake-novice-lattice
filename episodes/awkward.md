---
title: Awkward corners
teaching: 20
exercises: 20
---

::::::::::::::::::::::::::::::::::::::: objectives

- Understand how Snakemake being built on Python allows us 
  to work around some shortcomings of Snakemake in some use cases.
- Understand how to handle trickier metadata and input file lookups.
- Understand how to handle nested Python directory structures

::::::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::: questions

- How can I look up metadata based on wildcard values?
- How can I select different numbers of input files
  depending on wildcard values?
- How can I have a rule depend on a Python source file
  that relies on importing files from subdirectories?

::::::::::::::::::::::::::::::::::::::::::::::::::

## Beyond the pseudoscalar

Recall the rule that we have been working on
to fit the correlation function of the pseudoscalar meson:

```source
rule ps_mass:
    input:
        data="raw_data/beta{beta}/out_corr",
    output:
        data="intermediary_data/beta{beta}/corr.ps_mass.json",
        plot="intermediary_data/beta{beta}/corr.ps_eff_mass.{plot_filetype}",
    log:
        messages="intermediary_data/beta{beta}/corr.ps_mass.log",
    params:
        plateau_start: lookup(within=metadata, query="beta = {beta}", cols="ps_plateau_start"),
        plateau_end: lookup(within=metadata, query="beta = {beta}", cols="ps_plateau_end"),
    conda: "../envs/analysis.yml"
    shell:
        "python -m su2pg_analysis.meson_mass {input.data} --output_file {output.data} --plateau_start {params.plateau_start} --plateau_end {params.plateau_end} --plot_file {output.plot} --plot_styles {plot_styles} |& tee {log.messages}"
```

Now,
we are frequently interested in more symmetry channels than just the pseudoscalar.
In principle,
we could make a copy of this rule,
and change `ps` to,
for example
`v` or `av`.
However,
just like adding more ensembles,
this rapidly makes our Snakefiles unwieldy and difficult to maintain.
How can we adjust this rule so that it works for any channel?

The first step is to replace `ps` with a wildcard.
The `output` should then use `corr.{channel}_mass.json`
rather than `_corr.ps_mass.json`,
and similarly for the log.
We will also need to pass an argument
`--channel {wildcards.channel}`
in the `shell` block,
so that the code knows what channel it is working with.
Since the rule is no longer only for the `ps` channel,
it should be renamed,
for example to `meson_mass`.

What about the plateau start and end positions?
Currently these are 

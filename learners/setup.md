---
title: Setup
---

## Software installation

These instructions set out how to obtain and install the software and data on Linux. It is
assumed that you have:

- access to the Bash or Zsh shell on a fairly modern Linux or macOS system
- sufficient disk space (~1GB) to store the software and data

You **do not** need root/administrator access.


## Data Sets

<!--
FIXME: place any data you want learners to use in `episodes/data` and then use
       a relative link ( [data zip file](data/lesson-data.zip) ) to provide a
       link to it, replacing the example.com link.
-->
Download the [data zip file](https://example.com/FIXME) and unzip it to your Desktop

## Software Setup

### Conda

We will use Conda both to install Snakemake itself,
and to manage dependencies of our workflows.
Miniforge provide a minimal Conda environment,
on which we will build.

::::::::::::::::::::::::::::::::::::::: discussion

### Details

Download the correct file for your operating system from
[the Miniforge repository][miniforge],
and execute it at the terminal.

:::::::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::: spoiler

### Windows

This lesson has not been tested with Windows.
We would recommend using the Windows Subsystem for Linux,
and following the instructions for Linux.

::::::::::::::::::::::::

:::::::::::::::: spoiler

### macOS, Linux

```bash
curl -L -O "https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-$(uname)-$(uname -m).sh"
bash Miniforge3-$(uname)-$(uname -m).sh
```

When prompted,
unless you have a reason not to,
pick the option for Conda to set up your environment with `conda init`.

::::::::::::::::::::::::

## Snakemake

With Conda available,
we can create an environment containing Snakemake and its dependencies.
This can be used not just for this lesson,
but for your work in Snakemake going forward.

::::::::::::::::::::::::::::::::::::::: discussion

### Details

```bash
conda create -n snakemake -c conda-forge -c bioconda snakemake
conda activate snakemake
conda install -c conda-forge 'mamba<2.0.0'
```

After starting a new terminal,
or rebooting your computer,
you will need to run

```bash
conda activate snakemake
```

in order to activate the environment to be able to use Snakemake.

:::::::::::::::::::::::::::::::::::::::::::::::::::


[miniforge]: https://github.com/conda-forge/miniforge

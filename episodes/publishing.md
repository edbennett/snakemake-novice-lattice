---
title: Publishing your workflow
teaching: 20
exercises: 20
---

::::::::::::::::::::::::::::::::::::::: objectives

- Understand the need to test a workflow
- Be able to package a workflow for upload

::::::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::: questions

- How do I verify that my workflow is ready to upload?
- How do I prepare a single archive of my workflow and its dependencies?

::::::::::::::::::::::::::::::::::::::::::::::::::

So far,
we've talked about how to develop a workflow
reliably and consistently reproduces the same results.
But at some point we would like to publish those results.
The scientific method demands that we enable others to reproduce our work.
While in principle,
they might be able to do this just from
the equations and references we write in our paper,
in practice we can never provide enough detail to encode every decision our code makes,
and so we must publish our workflow to enable reproducibility by others.

In the previous episode,
we discussed keeping the workflow in Git.
While Git repositories can be made accessible to others through services like GitHub,
this is only really suitable for code that is under active development;
for long term,
static retention,
it is better to use a service dedicated to providing that.

In particular,
you want references to the workflow in publications to be accessible
arbitrarily far in the future.
If I see a citation in a paper to a paper from 1955,
I can go to the library and read it,
or find it on the journal's website.
Most services designed for active use can't provide guarantees of longevity;
even if the resources remain available,
their address might change.
For ongoing work,
this is a minor inconvenience,
but for a published paper,
we can't reasonably go back and edit every citation to every workflow years later.

For that reason,
data, software, and workflows
should be published in an appropriate data or repository.
In some fields there are specific repositories focusing on the needs of that community;
in lattice,
currently there is none.
However,
for sharing most workflows and modest data
(up to 50GB,
or 200GB on specific request),
a suitable general-purpose repository is [Zenodo][zenodo].
This is run by CERN,
and is committed to remain available for the life of the CERN datacentre.
It provides a DOI for each uploaded dataset,
which can be cited in papers in the same way as the DOI for a journal article.

## Testing the workflow

Before releasing the workflow to the world,
it's important to verify that it does correctly reproduce the expected outputs.
Since you will have been using it for this in development,
this would hopefully be automatic,
but you can find situations where existing files left over from earlier in development
allow the workflow to complete in your working directory,
but not in a freshly-cloned version.

Let's use Git to make a fresh copy of the workflow:

```shellsession
cd ../
git clone --recurse-submodules su2pg su2pg_test
cd su2pg_test
```

Now,
we follow the steps in the README directly.
We obtain the data and metadata,
and place them into the correct locations.
In practice we would want to have the data
already in the ZIP format that we will use to upload to Zenodo;
for now,
we can copy this across:

```shellsession
cp -r ../su2pg/raw_data .
cp ../su2pg/metadata/* metadata/
```

Now we can re-run Snakemake:

```shellsession
snakemake --cores all --use-conda
```

Since this is a clean copy,
Snakemake will once again need
to install the Conda environment used by some of the workflow steps.

If Snakemake exits without errors,
we can do a quick sense check that the outputs we get match the ones we expect;
for example,
looking at plots side-by-side.
For text files,
we can also check the differences explicitly:

```shellsession
diff -r ../su2pg/assets/tables assets/tables
```

We expect to see differences in provenance,
but not in numbers,
when we run on the same computer.

:::::::::::::::::::::::::::::::::::::::: callout

If you can,
it's a good idea to test a different architecture to the one you developed on,
and to get a collaborator unfamiliar with the workflow to test the instructions.
For example,
if you developed the workflow on Linux,
then get a collaborator with a Mac to follow the README without your direct guidance.

This will likely raise some issues.
In some cases
the Conda environment will need to be adjusted to be cross-platform compatible.
Note that
numerical outputs are _not_ expected to be identical to machine precision
when switching between different platforms,
due to differences in numerics between different CPUs.
However,
there should be no visible differences in plots,
and tables should be consistent to the last printed significant figure.

::::::::::::::::::::::::::::::::::::::::::::::::

## Preparing a ZIP file

Since we cannot upload directories to Zenodo,
we must prepare a ZIP archive of the full set of code.
While GitHub does provide a ZIP version of repositories,
it excludes the contents of submodules,
instead leaving an empty directory,
which is not very useful to potential readers.
Instead,
we must do this preparation by hand.

::::::::::::::::::::::::::::::::::: instructor

In principle,
another archive like `tar.gz` could also be used.
However,
Zenodo can preview the contents of ZIP files,
and not of other archives,
so ZIP is strongly recommended here.

::::::::::::::::::::::::::::::::::::::::::::::

We again start from a clean clone,
ensuring that we have all submodules present
(since they may not be available from GitHub later).

```shellsession
cd ..
git clone --recurse-submodules su2pg su2pg_release
zip -9 --exclude "**/.git/*" --exclude "**/.git" -r su2pg.zip su2pg_release
```

Note that we use `--exclude` to avoid
committing the `.git` directory of the repository or its submodules.
We don't get the `.git` directory in a GitHub ZIP file either,
and the reader doesn't need it:
they only require the final version used to generate the work in the paper,
not the full history.

At this point,
you should be ready to upload to Zenodo.
Completing the upload form is beyond the scope of this lesson,
but more detail can be found in the
[TELOS Collaboration reproducibility and open science guide][telos-guide],
or in the lesson [Publishing your Data Analysis Code][publishing-analysis].

:::::::::::::::::::::::::::::::::::::::: keypoints

- Test the workflow by making a fresh clone and following the README instructions
- Use `zip -9 --exclude "**/.git/*" --exclude "**/.git" filename.zip dirname`
  to prepare a ZIP file of a freshly-cloned repository,
  with submodules.

::::::::::::::::::::::::::::::::::::::::::::::::::

[publishing-analysis]: https://edbennett.github.io/publishing-analysis/08-pushing/index.html
[telos-guide]: https://github.com/telos-collaboration/strategy
[zenodo]: https://zenodo.org

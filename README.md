# slurm-goodies

Slurm job definitions and wrapper scripts to make (some) bioinformatician
lives easier.


## Background

Slurm lacks native support for some higher level functions that are
daily chores for bioinformaticians:

 * Iterate a batch job over a collection of inputs (e.g. fasta files)
 * Run a workflow of dependent jobs (and do this across a batch)

Additionally, having a library of "pre-fab" jobs for common tasks would
be convenient.  This repository aims to fulfil these three goals.

#### Batch support

Basic support for batching is present in Slurm: `sbatch --array` creates
a batch of parallel jobs with identical requirements.  However, it is not
as trivially usable as these HTCondor examples:

    # Create array of jobs, one per matching file $F
    queue [a job for each] F matching /path/to/*.fasta

    # Create array of jobs that take params IN, OUT from a TSV file
    queue [a job for each] IN,OUT from table.tsv

The `sbatch-list` script in this repo intends to fill this gap.

#### Workflow support

The `--dependency` option for jobs in Slurm provides a basis for building
workflows, by enabling jobs to wait for completion statuses of other jobs
(including array jobs, using the `aftercorr` condition).

However, it is far from a simple high-level specification, like:

    # Define a miniature "diamond-shaped" workflow (HTCondor DAG)
    PARENT jobA       CHILD jobB jobC
    PARENT jobB jobC  CHILD jobD

Dedicated workflow engines such as NextFlow may be the best option here,
to the extent that they interoperate with Slurm.

> It is pointed out [here](https://groups.google.com/g/slurm-users/c/7ySh6mJt9so)
> that NextFlow (in 2023) did not (yet) support Slurm arrays.
>
> The [MyQueue](https://myqueue.readthedocs.io/en/latest/) front-end
> linked from that page offers workflow support, albeit using Python.
>
> **TODO** investigate these and other options for workflow support
> (which is not quite as trivial as batch support: partial success,
> resuming runs, etc.)

#### Job Library

The `lib/jobs` directory is intended as a repository of reusable
bioinformatics jobs.

An elegant feature of Slurm is that job files are plain shell scripts,
which can (using `#SBATCH` directives) be annotated with their resource
requirements.

Once you have the job file, running e.g. a SKESA assembly can be simply:

    sbatch lib/jobs/skesa.job read1.fq read2.fq mysample.fna

... with parameter settings, input checks, usage information _and_ Slurm
resource requests all captured as "presets" in the job file.

If written properly, job files (being shell scripts) can be self-contained:

    $ lib/jobs/skesa.job --help
    Usage: ... READS1 READS2 OUT_DIR
    ...

And finally, such job files can be used as-is by `sbatch-list` (below) to
process a batch of inputs:

    sbatch-list inputs.lst lib/jobs/skesa.job

> We may choose to move some typical boilerplate to a central function
> library, though in principle job files should best be 'decoupled'.

#### Container support

**TODO** investigate Slurm's support for running OCI containers (Docker,
Singularity).  For now we run these as we would normally (`docker run ..`)
with the permissions issues this entails.


## Usage

For now we have just the `batch-list` script for "batching a job" across
a list of inputs (such as all files in a directory).
 
### sbatch-list

The `sbatch-list` script takes a Slurm job file (or any plain bash script
or executable) and a text file, and creates an array of jobs, each getting
its own line from the text file as its command-line arguments:

    # Submits a job array, one job per line from LIST_FILE
    sbatch-list [SBATCH_OPTS] LIST_FILE|- JOB_FILE

When `LIST_FILE` is `-`, reads the list from stdin.  Empty lines and
comment lines (starting with `#`) are ignored.

Any `SBATCH_OPTS` are passed on to `sbatch` (but could better be specified
through `#SBATCH` directives in `JOB_FILE`).

#### Example: single command-line

This gzips (in parallel jobs across the cluster) all files in PWD:

    ls -Q * | sbatch-list - gzip

Note the `-Q` for quoted output (to protect file names with spaces),
and of course we might also want to prevent double-zipping:

    ls -Q * | grep -Ev '.gz"$' | sbatch-list - gzip

But will still fail if PWD contains a directory (exercise for the reader).

#### Example: list file and job file

A more elaborate example: let's batch SKESA over a collection of reads,
listed (with their isolate ID) in file `reads.lst`:

    File: reads.lst
    ------------------------------------------------------------------
    # isolate    # reads1               # reads2
    sample1.fna  /path/to/reads1_R1.fq  /path/to/reads1_R2.fq
    sample2.fna  /path/to/reads2_R1.fq  /path/to/reads2_R2.fq
    ...

Here it makes sense to first write a job script that assembles a single
read pair:

    File: skesa.job  (error handling omitted)
    ------------------------------------------------------------------
    #/bin/bash
    #SBATCH -c 8 --mem 32G -t 30

    # We expect three arguments: sample ID, path to R1, path to R2
    FNA="$1.fna" R1="$2" R2="$3"

    # Invoke SKESA, matching our requested resources
    skesa --cores 8 --memory 32 --reads "$R1,$R2" --contigs_out "$FNA"
    ------------------------------------------------------------------

We can test this either outside of Slurm by running it directly:

    chmod +x skesa.job
    ./skesa.job sample1 /path/to/R1.fq /path/to/R2.fq

or submit it as a Slurm job with `sbatch`:

    sbatch skesa.job sample1 /path/to/R1.fq /path/to/R2.fq

And if all works well, let `sbatch-list` run it over the list:

    sbatch-list reads.lst skesa.job

##### Note: output handling

@TODO@: options `-o` and `-e`, and using `SBATCH_OPTS` to override

##### Note: improving the job file

The `skesa.job` above can be improved in several ways:

 * Provide `-h/--help`
 * Hardened bash: `export LC_ALL="C"; set -euo pipefail`
 * Check that inputs exist, else error out
 * Check that output does not exist, or:
   * Overwrite iff output is older than inputs, or
   * Add a `-f/--force` parameter, or
   * ...
 * Replace hardcoded params by `SLURM_*` variables
   * But remember that no bash commands (not even variable assignments)
   can come before the `#SBATCH` directives


### sbatch-tsv

**Work in Progress**

The `sbatch-tsv` script is like `sbatch-list` but parses the inputs for
the array jobs from a TSV file, and makes them available to the script
through environment variables.



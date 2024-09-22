# slurm-goodies

Slurm job definitions and wrapper scripts to make (some) bioinformatician
lives easier.


## Background

Slurm lacks native support for higher level functions that are daily
chores for bioinformaticians:

 * Iterate a batch job over a collection of inputs (e.g. fasta files)
 * Run a workflow of dependent jobs (and do this across a batch)
 * Maintain a library of jobs and workflows for common tasks

This repository aims to partially fill these gaps, either with tooling
or pointers to documentation.

### Batch support

Slurm has basic support for batching: `sbatch --array` creates an array
of parallel jobs with identical requirements.  However, this is not as
trivially usable as these HTCondor examples:

    # Create array of jobs, one per matching file $F
    queue [this job for each] F matching /path/to/*.fasta

    # Create array of jobs that take params IN, OUT from a TSV file
    queue [this job for each] IN, OUT from table.tsv

The `sbatch-list` script in this repo intends to fill part of this gap.

### Workflow support

Slurm has rudimentary workflow support: the `--dependency` option enables
jobs to wait for completion statuses of other jobs (including array jobs,
using the `aftercorr` condition).

However, this is nowhere near the functionality offered by e.g. HTCondor
DAGs or standalone workflow engines such as NextFlow or Cromwell.  For
instance, this specification:

    # Define a miniature "diamond-shaped" workflow
    PARENT jobA       CHILD jobB jobC
    PARENT jobB jobC  CHILD jobD

... integrates 4 jobs (of which 2 can run in parallel) in a workflow that
is directly executable on HTCondor, and can be managed like any other job:
suspend, abort, recover from error, etc.

The best option on Slurm is probably having a dedicated workflow engine,
and preferably one that integrates well.

It is pointed out [here](https://groups.google.com/g/slurm-users/c/7ySh6mJt9so)
that NextFlow (in 2023) did not (yet) support Slurm arrays.

The [MyQueue](https://myqueue.readthedocs.io/en/latest/) front-end
linked from that page offers workflow support, albeit using Python.

> I'm curious if NextFlow and Slurm can ever integrate well: a _workload_
> manager and a _workflow_ manager running a ship together?  So who is in
> charge of what happens when?  Who manages batches?  Who handles failure
> recovery?

**TODO** investigate these and other options for workflow support

### Job Library

The `lib/jobs` directory is intended as a repository of reusable
bioinformatics jobs.

An elegant feature of Slurm is that job files are plain shell scripts,
which can (using `#SBATCH` directives) be annotated with their resource
requirements.

Once you have the job file, running e.g. a SKESA assembly can be simply:

    sbatch lib/jobs/skesa.job read1.fq read2.fq mysample.fna

... with parameter settings, input checks, usage information _and_ Slurm
resource requests all captured as "presets" in the job file.

Job files (being shell scripts) can even be run directly:

    $ lib/jobs/skesa.job --help
    Usage: ... READS1 READS2 OUT_DIR
    ...

And finally, such job files can be used as-is by `sbatch-list` (below) to
process a batch of inputs:

    sbatch-list inputs.lst lib/jobs/skesa.job

> We may choose to move some typical boilerplate to a central function
> library, though in principle job files should best be 'decoupled'.

#### Note: container support

**TODO** investigate Slurm's support for running OCI containers (Docker,
Singularity, LXC, ...).  For now we run these as we would otherwise,
from a script that invokes `docker run ..`.  Permission issues may occur.


## Usage

For now we have the `batch-list` script for "batching a job" across a list
of inputs (such as all files in a directory), and `batch-tsv` for batching
a job across the 'records' in a TSV file.
 
### sbatch-list

The `sbatch-list` script takes a Slurm job file (or any plain bash script
or executable) and a text file, and creates an array of jobs, each getting
its own line from the text file as its command-line arguments:

    # Submits a job array, one job per line from LIST_FILE
    sbatch-list [SBATCH_OPTS] LIST_FILE|- JOB_FILE

When `LIST_FILE` is `-`, reads the list from stdin.  Empty lines and
comment lines (starting with `#`) are ignored.

Any `SBATCH_OPTS` are passed on to `sbatch`, and override any `#SBATCH`
directives in `JOB_FILE` (as with `sbatch`).

#### Example: single command-line

This gzips (in parallel jobs across the cluster) all files in PWD:

    ls -Q * | sbatch-list - gzip

The `-Q` makes `ls` quote output, so file names with spaces work as well.

If you'd want to pass additional arguments to gzip, these need to be added
to the command-line, e.g. like this:

    ls -Q * | sed 's/^/--best /' | sbatch-list - gzip

Note that it isIt is not possibly

#### Example: list file and job file

A more elaborate example: let's batch SKESA over a collection of reads,
listed in file `reads.lst`:

    File: reads.lst
    ------------------------------------------------------------------
    # isolate    # reads1               # reads2
    sample1      /path/to/reads1_R1.fq  /path/to/reads1_R2.fq
    sample2      /path/to/reads2_R1.fq  /path/to/reads2_R2.fq
    ...

We write a job script that calls SKESA, taking the names of the two reads
files and the assembly on its command-line:

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

And if all works well, let `sbatch-list` run it over our list:

    sbatch-list reads.lst skesa.job

##### Note: output handling

@TODO@: options `-o` and `-e`, and using `SBATCH_OPTS` to override

##### Note: improving the job file

The `skesa.job` above can be improved in several ways.  See the final
result in the job library: [lib/jobs/skesa.job](lib/jobs/skesa.job).


### sbatch-tsv

**Work in Progress**

The `sbatch-tsv` script is like `sbatch-list` but parses the inputs for
the array jobs from a TSV file, and makes them available to the script
through environment variables.


### List of job library

Currently in [lib/jobs](lib/jobs):

 * [skesa.job](lib/jobs/skesa.job)
 * **WIP** [bap.job](lib/jobs/bap.job)



---

### Licence

sbatch-goodies - Slurm jobs and wrapper scripts for bioinformaticians  
Copyright (C) 2024  Marco van Zwetselaar <io@zwets.it>

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program.  If not, see <http://www.gnu.org/licenses/>.

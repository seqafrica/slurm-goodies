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

The `sbatch-list` and `sbatch-tsv` scripts in this repo intend to fill
part of this gap.

### Workflow support

Slurm has rudimentary workflow support: the `--dependency` option enables
jobs to wait for completion statuses of other jobs (including array jobs,
using the `aftercorr` condition).

However, this is nowhere near the functionality offered by e.g. HTCondor
DAGs and especially workflow engines such as NextFlow or Cromwell.  For
instance, this specification:

    # Define a miniature "diamond-shaped" workflow
    PARENT jobA       CHILD jobB jobC
    PARENT jobB jobC  CHILD jobD

... integrates 4 jobs (of which 2 can run in parallel) in a workflow that
is directly executable on HTCondor, and can be managed like any other job:
suspend, abort, recover from error, etc.

#### Options on Slurm

The best option is probably to have a dedicated workflow engine that
integrates well with Slurm.

> I'm curious to see how this integration will work: a _workload_ manager
> and a _workflow_ manager ... feels a bit like two captains on a ship.
> Who is in charge of what happens when? Who controls batches? Where are
> failures handled?

It is pointed out [here](https://groups.google.com/g/slurm-users/c/7ySh6mJt9so)
that NextFlow integrates with Slurm but (in 2023) did not (yet) support
job arrays.

The [MyQueue](https://myqueue.readthedocs.io/en/latest/) front-end
linked from that page looks promising _and_ offers workflow support,
albeit using Python.

**TODO** investigate these and other options for workflow support.

### Job Library

The `lib/jobs` directory is intended as a repository of reusable
bioinformatics jobs.

An elegant feature of Slurm is that job files are plain shell scripts,
which can (using `#SBATCH` directives) be annotated with their resource
requirements.

Once you have the job file, running e.g. a SKESA assembly can be simply:

    sbatch lib/jobs/skesa.job reads1.fq reads2.fq mysample.fna

... with parameter settings, input checks, usage information _and_
Slurm resource requests all captured as "presets" in the job file.

Job files (being shell scripts) can even be run directly:

    $ lib/jobs/skesa.job --help
    Usage: ... READS1 READS2 OUT_DIR
    ...

And finally, such job files can be used as-is by `sbatch-list` (below)
to process a batch of inputs:

    sbatch-list inputs.lst lib/jobs/skesa.job

#### Note: container support

**TODO** investigate Slurm's support for running OCI containers (Docker,
Singularity, LXC, ...).  For now we run these as we would otherwise do:
`docker run --rm -it -u $(id -u):$(id -g) ...`.  **Note** this is likely
to present permissions issues.


## Usage

For now we have the `batch-list` script for "batching a job" across a list
of inputs (such as all files in a directory), and `batch-tsv` for batching
a job across all 'records' in a TSV file.
 
### sbatch-list

The `sbatch-list` script takes a Slurm job file (or any executable) and a
text file, and creates a job array, where each instance gets its own line
from the text file as its command-line arguments:

    sbatch-list [SBATCH_OPTS] LIST_FILE|- JOB_FILE

When `LIST_FILE` is `-`, reads the list from stdin.  Empty lines and
comment lines (starting with `#`) are ignored.

Any `SBATCH_OPTS` are passed on to `sbatch`, and (as with Slurm's `sbatch`)
override settings made in `#SBATCH` directives in `JOB_FILE`.

#### Example: single command-line

This gzips (in parallel jobs across the cluster) all `*.txt` files in PWD:

    ls -Q *.txt | sbatch-list - gzip

(The `-Q` makes `ls` quote its output, to deal with file names with spaces.)

If you want to pass additional arguments to gzip, you could inject these
in the command-line, e.g. like this:

    ls -Q *.txt | sed 's/^/--best /' | sbatch-list - gzip

For most jobs, it will be convenient to first create a list file and a job
file, as demonstrated in the next example.

#### Example: list file and job file

A more elaborate example: let's batch SKESA over a collection of reads,
listed in file `reads.lst`:

    File: reads.lst
    ------------------------------------------------------------------
    isolate1  /path/to/reads1_R1.fq  /path/to/reads1_R2.fq
    isolate2  /path/to/reads2_R1.fq  /path/to/reads2_R2.fq
    ...

We write a job script that calls SKESA, taking the isolate ID and the
names of the two reads files on its command-line:

    File: skesa.job  (error handling omitted)
    ------------------------------------------------------------------
    #/bin/bash
    #SBATCH -c 8 --mem 32G -t 30

    # We expect three arguments: sample ID, path to R1, path to R2
    ID="$1" R1="$2" R2="$3"

    # Invoke SKESA, matching our requested resources
    skesa --cores 8 --memory 32 --reads "$R1,$R2" --contigs_out "$ID.fna"
    ------------------------------------------------------------------

We can test this either outside of Slurm by running it directly:

    chmod +x skesa.job
    ./skesa.job isolate1 /path/to/R1.fq /path/to/R2.fq

or submit it as a Slurm job with plain `sbatch`:

    sbatch skesa.job isolate1 /path/to/R1.fq /path/to/R2.fq

or indeed batch it over all entries in reads.lst with `sbatch-list`:

    sbatch-list reads.lst skesa.job

This submits a job to `sbatch --array` that will invoke our `skesa.job`
once for each line from `reads.lst`.

> **Note** the `skesa.job` above has no error checking or other features.
> Look in the job library for a more elaborate version:
> [lib/jobs/skesa.job](lib/jobs/skesa.job).


### sbatch-tsv

`sbatch-tsv` is like `sbatch-list` in that it submits a job array with
one task for each line in a file.  In this case though, the lines are
not passed on the command-line, but through environment variables.

#### Example

Suppose we have this TSV file `inputs.tsv`:

| isolate | asm\_file    | fq1\_file            | ... | bap\_params     |
| :------ | :----------- | :------------------- | :-: | :-------------- |
| `iso1`  | `iso_01.fna` | `/path/to/id1_R1.fq` |  :  | `-v -t default` |
| `iso2`  | `iso_02.fna` | `/path/to/id2_R1.fq` |  :  | `-v ...`        |
| `...`   | `...`        | `...`                |  :  | `...`           |
|

> **Note** the TSV file must have a header line that defines the column
> names.  Column names must contain only letters, digits and underscores
> (so as to yield valid variable names).

In the job script, the value from each named column is available through
an environment variable with the same name, prefixed with an underscore:

    File: bap.job
    ------------------------------------------------------------------
    #/bin/bash
    #SBATCH -c 8 --mem 48G -t 60

    # Create output directory named after the isolate
    mkdir -p "$_isolate"

    # Call BAP with argument taken from the TSV columns
    BAP $_bap_params -o $_isolate $_fq1_file $_fq2_file
    ------------------------------------------------------------------

We can now submit the array job:

    sbatch-tsv inputs.tsv bap.job

This submits a job array with as many tasks as there are records in the
TSV file.  Each task invokes `bap.job` with its own set of values passed
through the `$_{col_name}` variables.

> **Note** the `bap.job` above lacks error checking, `-h` handling, etc.
> Adding `set -eu` to the script would already help a lot, as it makes
> bash exit on error, including the use of an undefined variable.

Note that we _can_ test the job without submitting it.  We can invoke
it from the command-line, with required environment variables set:

    _isolate=iso1 _bap_params=-v _fq1_file=... ./bap.ejob

Finally, note that jobs could be made to support both modes: command
line arguments and environment variables.  Examples are in the
[job library](lib/jobs) (**WIP**).


### Job library

Currently in [lib/jobs](lib/jobs):

 * [skesa.job](lib/jobs/skesa.job): assembly of Illumina reads with SKESA
 * [fastp1.job](lib/jobs/fastp1.job): run fastp on single reads
 * [fastp2.job](lib/jobs/fastp2.job): run fastp on paired-end reads

WIP:

 * [bap.job](lib/jobs/bap-fasta.job): run the BAP on anything
   * [bap-fasta.job](lib/jobs/bap-fasta.job): run the BAP on an assembly
   * [bap-nano.job](lib/jobs/skesa.job): run the BAP on Nanopore reads
   * [bap-illu.job](lib/jobs/skesa.job): run the BAP on Illumina reads

In the examples below we assume the job library has been deployed at
`/hpc/lib/jobs`, and that `sbatch-list` and `sbatch-tsv` scripts are on
`PATH`.

#### Example: fastp2 (PE reads)

To run fastp on a collection of paired end reads, create a list file:

    fastp.list
    ---------------------------------------------------------------------
    # Arguments as for fastp2.job (see: /hpc/lib/jobs/fastp2.job --help)
    # FASTP_OPTIONS     READS1              READS2           OUTPREFIX
    -c -5 -3 -t 1 -l 64 /path/to/sam1_R1.fq /path/sam1_R2.fq outdir/iso1
    -c -5 -3 -t 1 -l 64 /path/to/sam2_R1.fq /path/sam2_R2.fq outdir/iso2
    -c -5 -3 -t 1 -l 64 /path/to/sam3_R1.fq /path/sam3_R2.fq outdir/iso3
    ---------------------------------------------------------------------

Now submit the job with

    sbatch-list fastq.list /hpc/lib/jobs/fastp2.job


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

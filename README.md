# slurm-goodies

Slurm job definitions and wrapper scripts to make (some) bioinformatician
lives easier.


## Rationale

Slurm is a great tool, but lacks native support for some "higher level"
functions that are daily chores for bioinformaticians:

 * Iterate a batch job over a collection of inputs (notably files)
 * Run a workflow of jobs that depend on each other (_and_ batch)

Basic support for horizontal batching is present: `sbatch --array` does
just this, however it is not as simple as it is with HTCondor:

    # Creates array of jobs, one per fasta file $F found
    queue F matching /path/to/*.fasta

    # Creates array of jobs, each taking variables from a TSV file
    queue IN,OUT from table.tsv

Also, basic workflow support is provided by the `--dependency` option,
which enables jobs to wait for termination of other jobs.  With the
`aftercorr` condition, this "generalises" to array jobs.

However, this is a far cry from a simple high-level definition like:

    # Define a miniature "diamond-shaped" workflow (HTCondor DAG)
    PARENT jobA       CHILD jobB jobC
    PARENT jobB jobC  CHILD jobD

> **Note**: workflow engines like NextFlow do support such dependencies,
> and we need to look at their support for/by Slurm.
>
> However, [here](https://groups.google.com/g/slurm-users/c/7ySh6mJt9so)
> it is mentioned that NextFlow (in 2023) does not support array jobs.
>
> The [MyQueue](https://myqueue.readthedocs.io/en/latest/) front-end
> linked from that page offers workflow support, but in Python code.


## sbatch-list

The `sbatch-list` script takes a plain bash script (or any executable)
and a text file, and creates an array of jobs, passing each job a line
from the text file as its command-line arguments:

    # Submits a job array, one job per line from LIST_FILE
    sbatch-list [SBATCH_OPTS] LIST_FILE|- EXECUTABLE

When `LIST_FILE` is `-`, reads the list from stdin.  Empty lines and
comment lines (starting with `#`) are ignored.

### Examples

This gzips all non-gz files in PWD (in parallel):

    ls -Q * | grep -Ev '.gz"$' | sbatch-list - gzip

A more elaborate example: to batch SKESA over a collection of reads,
listed in file `reads.lst`:

    File: reads.lst
    ------------------------------------------------------------------
    # assembly   # reads1               # reads2
    sample1.fna  /path/to/reads1_R1.fq  /path/to/reads1_R2.fq
    sample2.fna  /path/to/reads2_R1.fq  /path/to/reads2_R2.fq
    ...

First write a shell script (e.g. `skesa.job`) that works for a single line:

    File: skesa.job
    ------------------------------------------------------------------
    #/bin/bash
    #SBATCH -c 8 --mem 32G -t 15

    # We expect three arguments: sample ID, path to R1, path to R2
    FNA="$1.fna" R1="$2" R2="$3"

    # Invoke SKESA, matching our requested resources
    skesa --cores 8 --memory 32 --reads "$R1,$R2" --contigs_out "$FNA"
    ------------------------------------------------------------------

Then test `skesa.job` against a single sbatch:

    sbatch skesa.job sample1 /path/to/R1 /path/to/R2

And if that works, let sbatch-list create the array for all:

    sbatch-list reads.lst skesa.job


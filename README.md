
# Snakemake dev

The critical part of snakemake workflow logic are filenames, that are defined as input and output files of various rules.
Snakemake builds from such defined rules a DAG to produce the target filenames.

## Repository structure

```none
├── config
│   └── config.yaml     # Configuration file with which we will allow the user to configurate the workflow
├── test
│   └── fastqs
│       ├── test1_R1.fastq.gz   # Some test data
│       ├── test1_R2.fastq.gz
│       ├── test2_R1.fastq.gz
│       └── test2_R2.fastq.gz
└── workflow
    ├── envs    # Conda environment recipes for Snakemake rules
    │   ├── cutadapt.yaml
    │   └── seqtk.yaml
    └── Snakefile   # For now, all workflow logic is here
```

Check `workflow/Snakefile` to see rule definitions. In this example there are 2 rules to subsample and trim reads. These rules cooperate with `config/config.yaml` and `workflow/envs` where conda recipes are located for rules.

## Snakemake Installation

Simply install it using conda/mamba. It is preferred to use a frozen version of snakemake. Later we will use pre_commit and peppy:

```sh
mamba create -c conda-forge -c bioconda --name snakemake_dev snakemake=7.25 snakemake-wrapper-utils pre_commit peppy
```

Other tools that will be used in the workflow will be installed and managed by snakemake directly.

## Running

It is good to first just run dry-run:

```sh
snakemake --snakefile workflow/Snakefile --use-conda -c1 -n
```

Notes:
`--snakefile workflow/Snakefile` is default so it is not required to be provided.
`-c1` to run with 1 core.
`-n` is a short for dry-run. Snakemake will try to build the DAG of rules to run the target rule or target output
`--use-conda` tells snakemake to run rules using specified conda environments. Snakemake installs them for you.

After running there should be an **error**. All rules have wildcards and Snakemake cannot infer values for them. By providing a valid target, Snakemake can infer wildcards and correctly build the DAG.

In the following iterations, we will clear up some concepts around snakemake.

### Iteration no.1

The output of subsampling is defined as `results/subsampled/{sample}_R1.fastq.gz`. We have 2 samples: `test1` and `test2`. Let's see what happens when you define `results/subsampled/test1_R1.fastq.gz` as the target:

```sh
snakemake --snakefile workflow/Snakefile --use-conda -c1 -n results/subsampled/test1_R1.fastq.gz
```

Now there should be no errors and you can see that one job was produced:

```sh
rule subsample_reads:
    input: test/fastqs/test1_R1.fastq.gz, test/fastqs/test1_R2.fastq.gz
    output: results/subsampled/test1_R1.fastq.gz, results/subsampled/test1_R2.fastq.gz
    log: logs/subsampling/test1.log
    jobid: 0
    reason: Missing output files: results/subsampled/test1_R1.fastq.gz
    wildcards: sample=test1
    resources: tmpdir=/tmp

Job stats:
job                count    min threads    max threads
---------------  -------  -------------  -------------
subsample_reads        1              1              1
total                  1              1              1
```

You can see that `subsample_reads` rule was requested by snakemake as we declared to want to produce the target `results/subsampled/test1_R1.fastq.gz`. From this it was further inferred what input should be.

You can also notice that `trim_reads` rule is not there, as the output of our target rule `subsample_reads` does not depend on `trim_reads` output.

### Iteration no.2

Now let's define as target the trimmed output:

```sh
snakemake --snakefile workflow/Snakefile --use-conda -c1 -n results/trimmed/test1_R1.fastq.gz
```

There should be now 2 jobs, first to produce the subsampled reads and then to produce the trimmed reads:

```sh
rule subsample_reads:
    input: test/fastqs/test1_R1.fastq.gz, test/fastqs/test1_R2.fastq.gz
    output: results/subsampled/test1_R1.fastq.gz, results/subsampled/test1_R2.fastq.gz
    log: logs/subsampling/test1.log
    jobid: 1
    reason: Missing output files: results/subsampled/test1_R2.fastq.gz, results/subsampled/test1_R1.fastq.gz
    wildcards: sample=test1
    resources: tmpdir=/tmp

rule trim_reads:
    input: results/subsampled/test1_R1.fastq.gz, results/subsampled/test1_R2.fastq.gz
    output: results/trimmed/test1_R1.fastq.gz, results/trimmed/test1_R2.fastq.gz, results/trimmed/test1.qc.txt
    log: logs/cutadapt/test1.log
    jobid: 0
    reason: Missing output files: results/trimmed/test1_R1.fastq.gz; Input files updated by another job: results/subsampled/test1_R2.fastq.gz, results/subsampled/test1_R1.fastq.gz
    wildcards: sample=test1
    resources: tmpdir=/tmp

Job stats:
job                count    min threads    max threads
---------------  -------  -------------  -------------
subsample_reads        1              1              1
trim_reads             1              1              1
total                  2              1              1

```

You can see in action how wildcards "bubbled" from the target.
By requesting `results/trimmed/test1_R1.fastq.gz` snakemake inferred that rule `trim_reads` is needed, and further it now requests also `results/subsampled/test1_R1.fastq.gz`, thus inferring that `subsample_reads`. To clear up, this does not happen by transferring wildcards of one rule to the other, but simply by filenames; i.e. wildcards in rules do not have to match, although it is good practice that if they represent the same, they should have the same name.

### Final iteration

As you probably noticed, in previous iterations we requested only for one sample. It is also possible to request for targets for more samples in the same manner:

```sh
snakemake --snakefile workflow/Snakefile --use-conda -c1 -n results/trimmed/test1_R1.fastq.gz results/trimmed/test2_R1.fastq.gz
```

In practice this is not feasible as user does know the outputs (In development it can still be useful to test a part of the workflow).

Instead of manually declaring the target outputs, we can write a new rule, that will request all outputs as inputs. If we write the rule as the first rule, it is automatically declared as the target rule and its outputs as target outputs. See now `workflow/Snakefile2`.

```py
# The first rule is the target rule. Commonly named as `all`
rule all:
    # as inputs specify all the files that you want to produce.
    # You cant use wildcards here as there will be no output.
    # The only dynamic part in our workflow is sample
    # **for simplicity** we here declare a list of samples
    # although in practice it is not feasible (more on that later)
    input:
        # we use expand function to generate all possible combinations of samples and outputs
        expand("results/trimmed/{sample}_R1.fastq.gz", sample=["test1", "test2"]),
```

I think that this should clear up one of the main concepts of snakemake - DAG construction from file names and wildcard inferring.

Check the output now by running:

```sh
snakemake --snakefile workflow/Snakefile2 --use-conda  -c1 -n
```

## Running for real

To run for real, simply drop the `-n` argument:

```sh
snakemake --snakefile workflow/Snakefile2 --use-conda  -c1
```

You can see, that first conda environments will be installed and then jobs will proceed.
You can check results on the paths defined by rules.

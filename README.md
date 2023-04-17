# ![nf-core/pangenome](docs/images/nf-core-pangenome_logo.png)

[![GitHub Actions CI Status](https://github.com/nf-core/pangenome/workflows/nf-core%20CI/badge.svg)](https://github.com/nf-core/pangenome/actions)
[![GitHub Actions Linting Status](https://github.com/nf-core/pangenome/workflows/nf-core%20linting/badge.svg)](https://github.com/nf-core/pangenome/actions)
[![Nextflow](https://img.shields.io/badge/nextflow-%E2%89%A520.10.0-brightgreen.svg)](https://www.nextflow.io/)

[![install with bioconda](https://img.shields.io/badge/install%20with-bioconda-brightgreen.svg)](https://bioconda.github.io/)
[![Docker](https://img.shields.io/docker/automated/nfcore/pangenome.svg)](https://hub.docker.com/r/nfcore/pangenome)
[![Get help on Slack](http://img.shields.io/badge/slack-nf--core%20%23pangenome-4A154B?logo=slack)](https://nfcore.slack.com/channels/pangenome)

## Introduction

> **Warning:** This pipeline is currently UNDER CONSTRUCTION. Some features may not work or not work as intended! This pipeline can only be run with Nextflow version 22.04.5 or later.

<!-- TODO nf-core: Write a 1-2 sentence summary of what data the pipeline is for and what it does -->
**nf-core/pangenome** is a bioinformatics best-practise analysis pipeline for the rendering of a collection of sequences into a pangenome graph.
Its goal is to build a graph that is locally directed and acyclic while preserving large-scale variation. Maintaining local linearity is important for interpretation, visualization, mapping, comparative genomics, and reuse of pangenome graphs.

The pipeline is built using [Nextflow](https://www.nextflow.io), a workflow tool to run tasks across multiple compute infrastructures in a very portable manner. It comes with docker containers making installation trivial and results highly reproducible.

## Quick Start

 **Warning:** The Dockerfile Github Action is not running, yet. Therefore, make sure you always have the latest image. Another caveat is that you need to clone the repository before you can execute the pipeline. Once we have an automated docker image build on `nf-core`, these inconveniences will be gone.

1. Install [`nextflow`](https://nf-co.re/usage/installation)

2. Install any of [`Docker`](https://docs.docker.com/engine/installation/), [`Singularity`](https://www.sylabs.io/guides/3.0/user-guide/), [`Podman`](https://podman.io/), [`Shifter`](https://nersc.gitlab.io/development/shifter/how-to-use/) or [`Charliecloud`](https://hpc.github.io/charliecloud/) for full pipeline reproducibility _(please only use [`Conda`](https://conda.io/miniconda.html) as a last resort; see [docs](https://nf-co.re/usage/configuration#basic-configuration-profiles))_

3. Build the current docker image if necessary

   ```bash
   docker build --no-cache . -t nfcore/pangenome:dev
   ```

4. Test the workflow on a minimal dataset

    ```bash
    nextflow run nf-core/pangenome -profile test,<docker/singularity/podman/shifter/charliecloud/conda/institute> --n_haplotypes 12
    ```

    [//]: # (```bash nextflow run nf-core/pangenome -profile test,<docker/singularity/conda/institute>```)

    > Please check [nf-core/configs](https://github.com/nf-core/configs#documentation) to see if a custom config file to run nf-core pipelines already exists for your Institute. If so, you can simply use `-profile <institute>` in your command. This will enable either `docker` or `singularity` and set the appropriate execution settings for your local compute environment.

5. Start running your own analysis!

    ```bash
    nextflow run nf-core/pangenome -profile <docker/singularity/podman/shifter/charliecloud/conda/institute> --input "input.fa.gz" --n_haplotypes 12
    ```

Be careful, the input FASTA must have been compressed with [bgzip](http://www.htslib.org/doc/bgzip.html). See [usage docs](https://nf-co.re/pangenome/usage) for all of the available options when running the pipeline.

## Examples

### DRB1-3123 Pangenome Graph

For an optimal workload of the computations, we can specify the resources of each process in a config file. We can call this for example `savasana.config`:

```
process {
    withName:'wfmash|seqwish|odgiLayout|wfmashMap|wfmashAlign|vg_deconstruct|gfaffix' {
        cpus = 4
        memory = 4.GB
    }

    withName:'smoothxg|odgiDraw|odgiBuild' {
        cpus = 4
        memory = 4.GB
    }

    withName:'splitApproxMappingsInChunks|odgiStats|odgiViz|multiQC|odgiDraw' {
        cpus = 1
        memory = 1.GB
    }
}
```

Executing the pipeline using the config file:

```bash
nextflow run nf-core/pangenome -r dev -profile docker --input ~/git/pggb/data/HLA/DRB1-3123.fa.gz -c savasana.config --n_haplotypes 12 --wfmash_map_pct_id 70 --wfmash_segment_length 2000 --smoothxg_poa_length 2000
```

## Advantages over [`pggb`](https://github.com/pangenome/pggb)

This Nextflow pipeline version's major advantage is that it can distribute the usually computationally heavy all versus all alignment step across a whole cluster. It is capable of splitting the initial approximate alignments into problems of equal size. The base-level alignments are then distributed across several processes. Assuming you have a cluster with 10 nodes and you are the only one using it, we would recommend to set `--wfmash_chunks 10`.
If you have a cluster with 20 nodes, but you have to share it with others, maybe setting it to `--wfmash_chunks 10` could be a good fit, because then you don't have to wait too long for your jobs to finish.

## Building a native container

It has been evaluated, that the current PGGB docker image can lead to slow performance of certain processes. 
While a single process my run up to 10 times slower compared to a native build, performance increase of a native build is expected to be ~30% regarding a whole pipeline run. This can be critical when building very large pangenomes. The [native_image.sh](bin/native_image.sh) script can tackle this problem.

### Singularity

Assuming you are located in your `git` folder, where you have both the nf-core/pangenome and the PGGB repository, then just

```bash
cp pangenome/bin/native_image.sh
./native_image.sh
```

This will generate a new folder, `pangenome_image`, in which all relevant files to build a native singularity container will be stored. After the script is finished, you can use the `pangenome-dev.img` in a pipeline:

```bash
nextflow run nf-core/pangenome -r dev - profile singularity --input ~/git/pggb/data/HLA/DRB1-3123.fa.gz --n_haplotypes 12 -with-singularity PATH_TO/pangenome_image/pangenome-dev.img
```

### Docker

If you want to build a docker image directly, just take a look at the script itself and comment out all lines that are surrounded by `#### SKIP ... ####`.

Then you can also run `./native_image.sh`. You can then execute a pipeline:

```bash
nextflow run nf-core/pangenome -r dev -profile docker --input ~/git/pggb/data/HLA/DRB1-3123.fa.gz --n_haplotypes 12 -with-docker ${USER}/pangenome-dev:latest
```

## Pipeline Summary

<!-- TODO nf-core: Add a brief summary of what the pipeline does and how it works -->

## Documentation

The nf-core/pangenome pipeline comes with documentation about the pipeline: [usage](https://nf-co.re/pangenome/usage) and [output](https://nf-co.re/pangenome/output).

<!-- TODO nf-core: Add a brief overview of what the pipeline does and how it works -->

## Credits

nf-core/pangenome was originally adapted from the pangenome graph builder [`pggb`](https://github.com/pangenome/pggb) pipeline by Simon Heumos, Michael Heuer.

Many thanks to all who have helped out and contributed along the way, including (but not limited to)\*:

| Name                                                     | Affiliation                                                                           |
|----------------------------------------------------------|---------------------------------------------------------------------------------------|
| [Philipp Ehmele](https://github.com/imipenem)            | [Institute of Computational Biology, Helmholtz Zentrum München, Munich, Germany](https://www.helmholtz-muenchen.de/icb/index.html)         |
| [Gisela Gabernet](https://github.com/ggabernet)                  | [Quantitative Biology Center (QBiC) Tübingen, University of Tübingen, Germany](https://uni-tuebingen.de/en/research/research-infrastructure/quantitative-biology-center-qbic/)|
| [Erik Garrison](https://github.com/ekg)                  | [The University of Tennessee Health Science Center, Memphis, Tennessee, TN, USA](https://uthsc.edu/)|
| [Andrea Guarracino](https://github.com/AndreaGuarracino) | [Genomics Research Centre, Human Technopole, Milan, Italy](https://humantechnopole.it/en/)        |
| [Friederike Hanssen](https://github.com/FriederikeHanssen) | [Quantitative Biology Center (QBiC) Tübingen, University of Tübingen, Germany](https://uni-tuebingen.de/en/research/research-infrastructure/quantitative-biology-center-qbic/)        |
| [Michael Heuer](https://github.com/heuermh)              | [UC Berkeley, USA](https://rise.cs.berkeley.edu)                                      |
| [Lukas Heumos](https://github.com/zethson)               | [Institute of Computational Biology, Helmholtz Zentrum München, Munich, Germany](https://www.helmholtz-muenchen.de/icb/index.html) <br> [Institute of Lung Biology and Disease and Comprehensive Pneumology Center, Helmholtz Zentrum München, Munich, Germany](https://www.helmholtz-muenchen.de/ilbd/the-institute/cpc/index.html) |
| [Simon Heumos](https://github.com/subwaystation)         | [Quantitative Biology Center (QBiC) Tübingen, University of Tübingen, Germany](https://uni-tuebingen.de/en/research/research-infrastructure/quantitative-biology-center-qbic/) <br > [Biomedical Data Science, Department of Computer Science, University of Tübingen, Germany](https://uni-tuebingen.de/en/faculties/faculty-of-science/departments/computer-science/department/) |

> \* Listed in alphabetical order

## Contributions and Support

If you would like to contribute to this pipeline, please see the [contributing guidelines](.github/CONTRIBUTING.md).

For further information or help, don't hesitate to get in touch on the [Slack `#pangenome` channel](https://nfcore.slack.com/channels/pangenome) (you can join with [this invite](https://nf-co.re/join/slack)).

## Citations

<!-- TODO nf-core: Add citation for pipeline after first release. Uncomment lines below and update Zenodo doi. -->
<!-- If you use  nf-core/pangenome for your analysis, please cite it using the following doi: [10.5281/zenodo.XXXXXX](https://doi.org/10.5281/zenodo.XXXXXX) -->

You can cite the `nf-core` publication as follows:

> **The nf-core framework for community-curated bioinformatics pipelines.**
>
> Philip Ewels, Alexander Peltzer, Sven Fillinger, Harshil Patel, Johannes Alneberg, Andreas Wilm, Maxime Ulysse Garcia, Paolo Di Tommaso & Sven Nahnsen.
>
> _Nat Biotechnol._ 2020 Feb 13. doi: [10.1038/s41587-020-0439-x](https://dx.doi.org/10.1038/s41587-020-0439-x).

In addition, references of tools and data used in this pipeline are as follows:

> **ODGI: understanding pangenome graphs.**
>
> Andrea Guarracino*, Simon Heumos*, Sven Nahnsen, Pjotr Prins & Erik Garrison.
>
> _Bioinformatics_ 2022 Jul 01 doi: [10.1093/bioinformatics/btac308](https://doi.org/10.1093/bioinformatics/btac308).
>
> *_contributed equally_

> **Unbiased pangenome graphs**
>
> Erik Garrison, Andrea Guarracino.
>
> _Bioinformatics_ 2022 Nov 30 doi: [10.1093/bioinformatics/btac743](https://doi.org/10.1093/bioinformatics/btac743).

<!-- TODO nf-core: Add bibliography of tools and data used in your pipeline -->

## Attention

### MultiQC Report  

In the resulting MultiQC report, in the **Detailed ODGI stats table**, it says `smoothxg`. To be clear, these are the stats of the graph after polishing with `gfaffix`! Some tools were hardcoded in the ODGI MultiQC module, but hopefully this will be fixed in the future.

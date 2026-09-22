# Benchmark-project
Benchmarking MEGAHIT vs. SPAdes genome assemblers, with and without redundans post-processing, on the heterozygous S288C×YJM789 yeast hybrid ancestor (SRR5221375). Reproducible Snakemake pipeline with automated runtime/memory benchmarking, evaluated via QUAST and BUSCO against the reference genome.

## Setup

This project requires Miniconda and the following bioinformatics tools. Follow the official installation instructions for each:

- **Miniconda** — https://docs.conda.io/en/latest/miniconda.html
- **Snakemake** — https://snakemake.readthedocs.io/en/stable/getting_started/installation.html
- **MEGAHIT** — https://github.com/voutcn/megahit
- **SPAdes** — https://github.com/ablab/spades
- **redundans** — https://github.com/Gabaldonlab/redundans
- **QUAST** — https://github.com/ablab/quast
- **BUSCO** — https://busco.ezlab.org/
- **SRA Toolkit** (for downloading reads) — https://github.com/ncbi/sra-tools

Once Miniconda is installed, most of these tools can also be installed via Bioconda, for example:

```bash
conda install bioconda::megahit
conda install bioconda::spades
```

**Don't forget to create your enviroment!**

```bash
conda create -n project
```

redundans is not available via conda and must be installed from its GitHub repository (see link above). If you would like to install it in a conda environment you must run this command:

```bash
conda create -n redundans -c conda-forge -c bioconda python=3.10 redundans
​```

to install BUSCO in a conda enviroment you must run this command:

```bash
conda create -n busco_env -c conda-forge -c bioconda busco
```

to install QUAST in a conda enviroment you must run this command:
```bash
conda create -n quast_env -c conda-forge -c bioconda quast
```

to install SRA toolkit in a conda enviroment you must run this command:
```bash
conda create -n sra_env -c conda-forge -c bioconda sra-tools
```

to install SRA toolkit in a conda enviroment you must run this command:
```bash
conda create -n snakemake_env -c conda-forge -c bioconda -c nodefaults snakemake
```

**Note:** Note: You do not need to run the `create -n nameofyourenvironment` part if you already have an environment created — you can just do `conda activate nameofyourenvironment` and install the software inside the environment. However, I do recommend having a separate environment for each software.

## Data Accessibility

This project uses publicly available Illumina paired-end sequencing reads from the diploid S288C × YJM789 hybrid ancestor strain of *Saccharomyces cerevisiae* (SRA accession `SRR5221375`, BioProject [PRJNA369471](https://www.ncbi.nlm.nih.gov/bioproject/PRJNA369471)).

### Downloading the reads

Reads are downloaded using the SRA Toolkit (`prefetch` and `fasterq-dump`):
```bash
prefetch SRR5221375
fasterq-dump SRR5221375 --split-files -O reads/
```

This produces two paired FASTQ files in the `reads/` directory: `SRR5221375_1.fastq` and `SRR5221375_2.fastq`.

**Note:** raw coverage is very high (~200×+) relative to the ~12 Mb genome. Reads will be used at full coverage initially; subsampling to a lower, more typical coverage (e.g., via `seqtk sample`) may be applied later if runtime or resource constraints on the shared server make it necessary.

### Downloading the reference genome

The S288C reference genome is used for reference-based evaluation (QUAST, BUSCO) and is downloaded from NCBI:
```bash
datasets download genome accession GCA_000146045.2 --include genome
```

**Don't forget to unzip it!**
```bash
unzip ncbi_dataset.zip -d GCA_000146045.2
```

The genome FASTA will be inside at a path like GCA_000146045.2/ncbi_dataset/data/GCA_000146045.2/GCA_000146045.2_*_genomic.fna 

Run ```find GCA_000146045.2 -name "*.fna"``` to get the exact path once unzipped. It will be helpful for the future!
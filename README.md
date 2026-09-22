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

​```bash
conda install bioconda::megahit
conda install bioconda::spades
​```

**Don't forget to create your enviroment!**

​```bash
conda create -n project
​```

redundans is not available via conda and must be installed from its GitHub repository (see link above). If you would like to install it in a conda environment you must run this command:

​```bash
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

**Note:** You do not need to do run the "create -n nameofyourenvironemt" part if you have already an enviroment created, you can just do 
```bash
conda activate -n project
```
and install the softwares inside the enviroment. However, I do recomment having a separated enviroment for each software.
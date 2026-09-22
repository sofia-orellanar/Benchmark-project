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

\`\`\`bash
conda install bioconda::megahit
conda install bioconda::spades
\`\`\`

Note: redundans is not available via conda and must be installed from its GitHub repository (see link above).
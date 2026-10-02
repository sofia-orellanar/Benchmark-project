# Benchmark-project
Benchmarking MEGAHIT vs. SPAdes genome assemblers, with and without redundans post-processing, on the heterozygous S288C×YJM789 yeast hybrid ancestor (SRR5221375). Reproducible Snakemake pipeline with automated runtime/memory benchmarking, evaluated via QUAST and BUSCO against the reference genome.

## Setup

This project requires Miniconda and several bioinformatics tools. Some of these tools needed non-standard installation steps on this server due to a CPU compatibility issue (see "Known Issues" below). Follow this section exactly rather than a plain conda install for MEGAHIT and SPAdes.

- **Miniconda** — https://docs.conda.io/en/latest/miniconda.html
- **Snakemake** — https://snakemake.readthedocs.io/en/stable/getting_started/installation.html
- **MEGAHIT** — https://github.com/voutcn/megahit
- **SPAdes** — https://github.com/ablab/spades
- **redundans** — https://github.com/Gabaldonlab/redundans
- **QUAST** — https://github.com/ablab/quast
- **BUSCO** — https://busco.ezlab.org/
- **SRA Toolkit** (for downloading reads) — https://github.com/ncbi/sra-tools

**1. Install Miniconda**
Follow the official instructions - https://docs.conda.io/en/latest/miniconda.html

**2. Create the main project environment**
Most tools install cleanly via Bioconda:

```bash
conda create -n assembler-benchmark python=3.10 -y
conda activate assembler-benchmark
conda install -c bioconda -c conda-forge snakemake quast busco seqtk lastal miniasm minimap2 bwa last -y
```

**3. Compile MEGAHIT and SPAdes from source**
This server's CPU does not support AVX2, which the standard Bioconda builds of MEGAHIT and SPAdes require. Both packages must be compiled from source instead of installed via ```conda install```.

- SPAdes

```bash
cd ~
wget https://github.com/ablab/spades/releases/download/v4.3.0/SPAdes-4.3.0.tar.gz
tar -xzf SPAdes-4.3.0.tar.gz
cd SPAdes-4.3.0
./spades_compile.sh
```

- MEGAHIT
```bash
cd ~
git clone https://github.com/voutcn/megahit.git
cd megahit
git submodule update --init
mkdir build && cd build
cmake ..
make -j8
```

Then symlink both compiled binaries into the assembler-benchmark environment so the Snakefile can call them directly

```bash
ln -sf ~/SPAdes-4.3.0/bin/spades.py ~/miniconda3/envs/assembler-benchmark/bin/spades.py
ln -sf ~/SPAdes-4.3.0/bin/spades-hammer ~/miniconda3/envs/assembler-benchmark/bin/spades-hammer
ln -sf ~/SPAdes-4.3.0/bin/spades-core ~/miniconda3/envs/assembler-benchmark/bin/spades-core
ln -sf ~/SPAdes-4.3.0/bin/spades-ionhammer ~/miniconda3/envs/assembler-benchmark/bin/spades-ionhammer
ln -sf ~/SPAdes-4.3.0/bin/spades-bwa ~/miniconda3/envs/assembler-benchmark/bin/spades-bwa
ln -sf ~/SPAdes-4.3.0/bin/spades-gbuilder ~/miniconda3/envs/assembler-benchmark/bin/spades-gbuilder
ln -sf ~/SPAdes-4.3.0/bin/spades-gmapper ~/miniconda3/envs/assembler-benchmark/bin/spades-gmapper
ln -sf ~/SPAdes-4.3.0/bin/spades-kmercount ~/miniconda3/envs/assembler-benchmark/bin/spades-kmercount
ln -sf ~/SPAdes-4.3.0/bin/spades-truseq-scfcorrection ~/miniconda3/envs/assembler-benchmark/bin/spades-truseq-scfcorrection
ln -sf ~/megahit/build/megahit ~/miniconda3/envs/assembler-benchmark/bin/megahit
```

Verify both work (should print version numbers, not crash)

```bash
hash -r
spades.py --version
megahit --version
```

**4. Install redundans in its own evironment**
The manual GitHub-source install of redundans ran into a cascade of missing dependencies. Installing it via its dedicated Bioconda recipe, in its own environment, resolves the full dependency tree correctly

```bash
conda create -n redundans python=3.10 -y
conda activate redundans
conda install -c bioconda -c conda-forge redundans -y
redundans.py --version   # should print a version number
```

The Snakefile calls this installation directly by its full path (see scripts/snakefile), so the redundans environment does not need to be separately activated when running the pipeline, only assembler-benchmark does.

**5. Running the pipeline**

```bash
conda activate assembler-benchmark
# if conda activate fails with "Run 'conda init' before 'conda activate'",
# run this first: source ~/miniconda3/etc/profile.d/conda.sh
snakemake -s scripts/snakefile --configfile scripts/config.yaml -j 8
```

## Known Issues
**AVX2 incompatibility:**
This server's CPU lacks AVX2 support, which caused Bioconda's prebuilt MEGAHIT and SPAdes binaries to crash immediately with a SIGILL (illegal instruction) error, exit code -4. Compiling both tools from source (see Setup, step 3) resolves this, since the compiled binaries are built for the server's actual CPU instead of assuming AVX2 availability. Check your own CPU's supported instructions with:```grep -m1 flags /proc/cpuinfo | tr ' ' '\n' | grep -E "avx2|popcnt"```

**Redudans dependency chain:**
Manual ```git clone + ./INSTALL.sh``` install of redundans failed with a missing-module import error (fasta2homozygous) when called via a symlink, then with a cascade of missing external dependencies (lastal, miniasm, gfastats, meryl) when called directly. Installing via Bioconda in its own dedicated environment (Setup, step 4) avoids all of this.

**Tmux + conda:**
A fresh tmux session does not automatically inherit conda's shell activation hooks. If conda activate fails with CondaError: Run ```conda init``` before ```conda activate``` inside a new tmux pane, run ```source ~/miniconda3/etc/profile.d/conda.sh``` first.

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

## Downloading the reference genome

**S288C reference genome:** used for reference-based evaluation (QUAST, BUSCO) and is downloaded from NCBI
```bash
datasets download genome accession GCA_000146045.2 --include genome
```

**Don't forget to unzip it!**
```bash
unzip ncbi_dataset.zip -d GCA_000146045.2
```

The genome FASTA will be inside at a path like GCA_000146045.2/ncbi_dataset/data/GCA_000146045.2/GCA_000146045.2_*_genomic.fna 

Run ```find GCA_000146045.2 -name "*.fna"``` to get the exact path once unzipped. It will be helpful for the future!

**YJM789 reference genome:** used for reference-based evaluation (QUAST, BUSCO) and is downloaded from NCBI:
```bash
datasets download genome accession GCA_000181435.1 --include genome
```

**Don't forget to unzip it!**
```bash
unzip ncbi_dataset.zip -d GCA_000181435.1
```

Run ```find GCA_000181435.1 -name "*.fna"``` to get the exact path once unzipped. It will be helpful for the future!

**Note:** if `datasets: command not found`, download it directly: `curl -o datasets 'https://ftp.ncbi.nlm.nih.gov/pub/datasets/command-line/v2/linux-amd64/datasets' && chmod +x datasets` and run as `./datasets` instead.

## Updating the config (yaml) file
Once both references are downloaded, update `scripts/config.yaml` with the actual paths:

```yaml
reference_s288c: "GCA_000146045.2/ncbi_dataset/data/GCA_000146045.2/GCA_000146045.2_R64_genomic.fna"
reference_yjm789: "GCA_000181435.1/ncbi_dataset/data/GCA_000181435.1/GCA_000181435.1_ASM18143v1_genomic.fna"
```
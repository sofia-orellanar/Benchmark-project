# Benchmark-project
Benchmarking MEGAHIT vs. SPAdes genome assemblers, with and without redundans post-processing, on the heterozygous S288C×YJM789 yeast hybrid ancestor (SRR5221375). Reproducible Snakemake pipeline with automated runtime/memory benchmarking, evaluated via QUAST and BUSCO against the reference genome.

## Contents

- [Setup](#setup)
- [Known Issues](#known-issues)
- [Data Accessibility](#data-accessibility)
- [Usage](#usage)
- [Results](#results)

## Setup

This project requires Miniconda and several bioinformatics tools. Some of these tools needed non-standard installation steps on this server due to a CPU compatibility issue (see "Known Issues" below).

Versions used for the results in this repository:

| Tool | Version | Homepage |
|---|---|---|
| Python | 3.10 | https://www.python.org/ |
| Snakemake | 7.32.4 | https://snakemake.readthedocs.io/en/stable/getting_started/installation.html |
| MEGAHIT | 1.2.9 | https://github.com/voutcn/megahit |
| SPAdes | 4.3.0 | https://github.com/ablab/spades |
| redundans | 2.0.1 | https://github.com/Gabaldonlab/redundans |
| QUAST | 5.3.0 | https://github.com/ablab/quast |
| BUSCO | 6.1.0 | https://busco.ezlab.org/ |
| SRA Toolkit | any recent version | https://github.com/ncbi/sra-tools |
| seqtk | any recent version | https://github.com/lh3/seqtk |

**1. Install Miniconda**

Follow the official instructions: https://docs.conda.io/en/latest/miniconda.html

**2. Create the main project environment**

Most tools install cleanly via Bioconda:

```bash
conda create -n assembler-benchmark python=3.10 -y
conda activate assembler-benchmark
conda install -c bioconda -c conda-forge snakemake quast busco seqtk miniasm minimap2 bwa last -y
```

**3. Install MEGAHIT and SPAdes**

First check whether your CPU supports AVX2:

```bash
grep -m1 flags /proc/cpuinfo | tr ' ' '\n' | grep -E "avx2|popcnt"
```

- **If `avx2` is listed**, the standard Bioconda builds should work. Install them with `conda install -c bioconda -c conda-forge megahit spades -y` and skip to the verification step below.
- **If `avx2` is NOT listed** (as on the server used for this project), the Bioconda builds of MEGAHIT and SPAdes crash with an illegal-instruction error (see "Known Issues"). Compile both from source instead:

SPAdes:

```bash
cd ~
wget https://github.com/ablab/spades/releases/download/v4.3.0/SPAdes-4.3.0.tar.gz
tar -xzf SPAdes-4.3.0.tar.gz
cd SPAdes-4.3.0
./spades_compile.sh
```

MEGAHIT:

```bash
cd ~
git clone https://github.com/voutcn/megahit.git
cd megahit
git submodule update --init
mkdir build && cd build
cmake ..
make -j8
```

Then symlink the compiled binaries into the `assembler-benchmark` environment so the Snakefile can call them directly:

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

Verify both work (they should print version numbers, not crash):

```bash
hash -r
spades.py --version
megahit --version
```

**4. Install redundans in its own environment**

A manual GitHub-source install of redundans ran into a cascade of missing dependencies. Installing it from its dedicated Bioconda recipe, in its own environment, resolves the full dependency tree correctly:

```bash
conda create -n redundans python=3.10 -y
conda activate redundans
conda install -c bioconda -c conda-forge redundans -y
redundans.py --version   # should print a version number
which redundans.py       # copy this path for the next step
```

The Snakefile calls redundans directly by its full path, so the `redundans` environment does not need to be activated when running the pipeline; only `assembler-benchmark` does. **Update the `REDUNDANS` path at the end of `scripts/config.yaml` to the path printed by `which redundans.py` on your machine.**

**5. Install the SRA Toolkit** (for downloading the reads)

```bash
conda create -n sra -c bioconda -c conda-forge sra-tools -y
conda activate sra
fasterq-dump --version   # should print a version number
```

## Known Issues

**AVX2 incompatibility:**
This server's CPU lacks AVX2 support, which caused Bioconda's prebuilt MEGAHIT and SPAdes binaries to crash immediately with a SIGILL (illegal instruction) error, exit code -4. Compiling both tools from source (see Setup, step 3) resolves this, since the compiled binaries are built for the server's actual CPU instead of assuming AVX2 availability.

**redundans dependency chain:**
A manual `git clone` + `./INSTALL.sh` install of redundans failed with a missing-module import error (`fasta2homozygous`) when called via a symlink, then with a cascade of missing external dependencies (lastal, miniasm, gfastats, meryl) when called directly. Installing via Bioconda in its own dedicated environment (Setup, step 4) avoids all of this.

**tmux + conda:**
A fresh tmux session does not automatically inherit conda's shell activation hooks. If `conda activate` fails with `CondaError: Run 'conda init' before 'conda activate'` inside a new tmux pane, run `source ~/miniconda3/etc/profile.d/conda.sh` first.

**SPAdes fragmentation (open question):**
SPAdes (run with `--isolate`) produced a far more fragmented assembly than MEGAHIT on this dataset, reproducibly across two full runs. The working hypothesis (isolate mode skips read error correction, combined with very high coverage and a variable insert size) has not been tested directly. See [RESULTS.md](RESULTS.md).

## Data Accessibility

This project uses publicly available Illumina paired-end sequencing reads from the diploid S288C × YJM789 hybrid ancestor strain of *Saccharomyces cerevisiae* (SRA accession `SRR5221375`, BioProject [PRJNA369471](https://www.ncbi.nlm.nih.gov/bioproject/PRJNA369471)).

### Downloading the reads

Reads are downloaded using the SRA Toolkit (`prefetch` and `fasterq-dump`):

```bash
conda activate sra
prefetch SRR5221375
fasterq-dump SRR5221375 --split-files -O reads/
```

This produces two paired FASTQ files in the `reads/` directory: `SRR5221375_1.fastq` and `SRR5221375_2.fastq`.

**Note:** raw coverage is very high (~200×+) relative to the ~12 Mb genome. All results in this repository use the full, unsubsampled reads.

### Downloading the reference genomes

Both references are downloaded from NCBI with the NCBI Datasets command-line tool (`datasets`).

**Note:** if `datasets: command not found`, download it directly and run it as `./datasets` instead of `datasets` in the commands below:

```bash
curl -o datasets 'https://ftp.ncbi.nlm.nih.gov/pub/datasets/command-line/v2/linux-amd64/datasets'
chmod +x datasets
```

#### S288C reference genome

Used for reference-based evaluation (QUAST).

```bash
datasets download genome accession GCA_000146045.2 --include genome --filename s288c_dataset.zip
unzip s288c_dataset.zip -d GCA_000146045.2
```

The genome FASTA will be inside at a path like `GCA_000146045.2/ncbi_dataset/data/GCA_000146045.2/GCA_000146045.2_*_genomic.fna`. Run this to get the exact path (you will need it for the config file below):

```bash
find GCA_000146045.2 -name "*.fna"
```

#### YJM789 reference genome

Also used for reference-based evaluation (QUAST), so that real assembly errors can be told apart from expected strain divergence in the hybrid.

```bash
datasets download genome accession GCA_000181435.1 --include genome --filename yjm789_dataset.zip
unzip yjm789_dataset.zip -d GCA_000181435.1
find GCA_000181435.1 -name "*.fna"
```

### Updating the config (yaml) file

Once both references are downloaded, update `scripts/config.yaml` with the actual paths from the `find` commands above:

```yaml
reference_s288c: "GCA_000146045.2/ncbi_dataset/data/GCA_000146045.2/GCA_000146045.2_R64_genomic.fna"
reference_yjm789: "GCA_000181435.1/ncbi_dataset/data/GCA_000181435.1/GCA_000181435.1_ASM18143v1_genomic.fna"
```

## Usage

### Inputs

- Paired-end reads: `reads/SRR5221375_1.fastq` and `reads/SRR5221375_2.fastq` (see Data Accessibility)
- Reference genomes: S288C and YJM789 FASTA files (paths set in `scripts/config.yaml`)
- `scripts/config.yaml` also sets the number of replicates (default 3), threads (default 8), and the BUSCO lineage (`saccharomycetes_odb10`; BUSCO downloads it on first use, which needs internet access)

### Running the full pipeline

Always run from the repository root. First do a dry run, which lists the planned jobs without executing anything:

```bash
conda activate assembler-benchmark
# if conda activate fails with "Run 'conda init' before 'conda activate'",
# run this first: source ~/miniconda3/etc/profile.d/conda.sh
snakemake -s scripts/snakefile --configfile scripts/config.yaml -n
```

Then run it for real. The full run took roughly 7 hours on 8 threads, so use `tmux` or `screen` so it survives a dropped connection:

```bash
snakemake -s scripts/snakefile --configfile scripts/config.yaml -j 8
```

If a run is interrupted, re-run the same command and Snakemake resumes from the jobs that are missing.

### What each step runs

The pipeline runs these commands for you. The equivalent standalone versions (for rep1) are:

```bash
# MEGAHIT
megahit -1 reads/SRR5221375_1.fastq -2 reads/SRR5221375_2.fastq -t 8 -o results/megahit_rep1

# SPAdes (isolate mode)
spades.py --isolate -1 reads/SRR5221375_1.fastq -2 reads/SRR5221375_2.fastq -t 8 -o results/spades_rep1

# redundans (contig-reduction step only)
redundans.py -f results/megahit_rep1/final.contigs.fa -i reads/SRR5221375_1.fastq reads/SRR5221375_2.fastq \
    -o results/megahit_redundans_rep1 --noscaffolding --nogapclosing -t 8

# QUAST (reference-based evaluation)
quast.py results/megahit_rep1/final.contigs.fa -r <reference.fna> -o results/quast/megahit_vs_s288c -t 8

# BUSCO (completeness)
busco -i results/megahit_rep1/final.contigs.fa -o megahit --out_path results/busco -l saccharomycetes_odb10 -m genome -c 8
```

### Quick test on a small dataset

To check that the tools are installed correctly without a multi-hour run, subsample the reads (about 10x coverage) and run the two assemblers standalone:

```bash
conda activate assembler-benchmark
mkdir -p test_data test_output
seqtk sample -s100 reads/SRR5221375_1.fastq 500000 > test_data/test_1.fastq
seqtk sample -s100 reads/SRR5221375_2.fastq 500000 > test_data/test_2.fastq

megahit -1 test_data/test_1.fastq -2 test_data/test_2.fastq -t 8 -o test_output/megahit
spades.py --isolate -1 test_data/test_1.fastq -2 test_data/test_2.fastq -t 8 -o test_output/spades
```

Use the same seed (`-s100`) for both files so read pairs stay matched. Each assembler should finish in a few minutes and write `test_output/megahit/final.contigs.fa` and `test_output/spades/contigs.fasta`. At this coverage the assemblies will not be meaningful; the test only confirms the tools run end to end.

### Outputs

| Path | Contents |
|---|---|
| `results/{megahit,spades}_rep{1-3}/` | Assembly output (`final.contigs.fa` for MEGAHIT, `contigs.fasta` for SPAdes) |
| `results/{megahit,spades}_redundans_rep{1-3}/` | redundans output; the reduced assembly is `scaffolds.reduced.fa` |
| `results/quast/{condition}_vs_{s288c,yjm789}/report.tsv` | N50, L50, genome fraction, duplication ratio, misassemblies |
| `results/busco/{condition}/` | BUSCO short summary (complete / duplicated / fragmented / missing) |
| `benchmarks/*.tsv` | Wall-clock time (`s`), CPU time, and peak memory (`max_rss`) per run, written by Snakemake's `benchmark:` directive |

`{condition}` is one of `megahit`, `spades`, `megahit_redundans`, `spades_redundans`. The `results/` folder is gitignored because the files are large; the key numbers are summarized in [RESULTS.md](RESULTS.md).

## Results

See [RESULTS.md](RESULTS.md) for the final metrics and key findings.
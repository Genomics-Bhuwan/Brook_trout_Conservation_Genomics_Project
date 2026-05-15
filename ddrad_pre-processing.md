#### ddRADseq Pre-assembly Pipeline for the Brook trout Project
#### Step 1. fastqc (Quality check for the data using fastqc)
#####################################
##### Quality Check with Fastqc ##########
#####################################
##Always check the names of pools names to confirm they are not the same. See the difference in the name
##There would be files with Library names: 1_S1_Loo1 or 1_S1_L002 or GH-1_S1_L002 OR GH_1_s1_L002
##Copy them all in a single folder and run the fastqc module

#### Step 1. Run the fastqc. I have land 1-3 and 4-6.
#### Step 1. a. Run the Fastqx for the lane 4-6. Do the same process for the lane 1-3
```bash

#!/bin/bash
#SBATCH --job-name=4_6fastqc
#SBATCH --account=bio260092
#SBATCH --partition=shared      # Use 'shared' for smaller resource requests
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=2      # 2 CPUs is plenty for one FastQC process
#SBATCH --mem=10G                # 8GB is more than enough for FastQC
#SBATCH --time=40:00:00         # 2 hours is plenty (100h was the main error)
#SBATCH --array=0-5
#SBATCH --output=fastqc_%A_%a.out

module load fastqc/0.11.9

OUTDIR="/anvil/projects/x-bio260092/Brook_Trout_Project/Lane_4_6/NV-C1703/fastqc"
mkdir -p $OUTDIR

# Important: Make sure you are in the directory where the .fq.gz files are
# or provide the full path to the files here:
FILES=(/anvil/projects/x-bio260092/Brook_Trout_Project/Lane_4_6/NV-C1703/*.fq.gz)

TARGET_FILE=${FILES[$SLURM_ARRAY_TASK_ID]}

echo "Processing file: $TARGET_FILE"

# Run FastQC (using 2 threads to match cpus-per-task)
fastqc -t 2 -o $OUTDIR $TARGET_FILE

```

#### Step 2. Clone filtering 
- This step will identify and PCR clones that we have in our sequence data.
- This steps use https://catchenlab.life.illinois.edu/stacks/manual/#barcode to determine which format you have
- Please check what format of the data is usually. For armadillo project, it was inline-index. I choosed cause our data is Paired End and has inline barcode on the singe end and an index barcode.
- We also need to check the type of oligo that we have used in our data.
- We usually have the random oligo on the singe end R1 sequence which leads us to use the -inline_null option in the clone filter.

-  We are not running clone filtering because I donot have unique molecular identifiers(UMIs), clone_filter identifies clones soley by looking at the start and end positions of the reads.
-  The SbfI problem in single-digest RAD: Every single valid sequence starts at the excat same SbfI cutsite.
-   If two different fish DNA fragments naturally happen to be the same length, clone_filter will incorrectly flag them as PCR duplicates and delete one.
-   The data might lose 20-50% of the real biological data.


#######################
##### Demultiplexing #####
#######################
#### Steps 3 a. Demultiplex reads using the process_radtag command in STACKS
#The process_radtags command in STACKS is used for demultiplexing, quality filtering, and trimming of reads generated from RAD-seq or related experiments. It allows you to sort and separate reads by individual sample based on barcode/index sequences and perform initial quality control.
# This can be done individually file by file or by writing a shell script (preferred method for time efficiency)
# First we write a loop to create a directory to hold each pool of individuals
# This code says that for number 1 to 6 we make a directory names Dasy_1, Dasy_2, etc
# The Dasy_undet will be for the undetermined file
# The semi colon between each command allows us to run the for loop on one line but tells the computer to treat the commands as though they are on separate lines
# Your upper limit should be how many pools you have, for example, if you have 8 the 6 would be an 8

```bash
#!/bin/bash
#SBATCH -p batch
#SBATCH --job-name=1_3_Demux
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=14
#SBATCH --time=70:00:00
#SBATCH --array=0-2%1
#SBATCH --mail-type=BEGIN,END,FAIL
#SBATCH --mail-user=bistbs@miamioh.edu
#SBATCH -o logs/demux_%A_%a.out
#SBATCH -e logs/demux_%A_%a.err

set -euo pipefail

module load stacks-2.66 2>/dev/null

# ---------------- PATHS ----------------
DATA_DIR="/shared/jezkovt_bistbs_shared/Brook_trout_Project/Lane_1_3/NV-C1702"
OUT_BASE="/shared/jezkovt_bistbs_shared/Brook_trout_Project/Lane_1_3/Processed_Radtags_1_3"

mkdir -p "$OUT_BASE" logs

# ---------------- SAMPLE MAP ----------------
PLATES=("A" "B" "C")

PLATE=${PLATES[$SLURM_ARRAY_TASK_ID]}

R1="${DATA_DIR}/NV_C1702${PLATE}_CKDL260007507-1A_23K3H2LT3_L5_1.fq.gz"
R2="${DATA_DIR}/NV_C1702${PLATE}_CKDL260007507-1A_23K3H2LT3_L5_2.fq.gz"
BC="${DATA_DIR}/C1702${PLATE}_barcodes.tsv"

OUTDIR="${OUT_BASE}/Parsed_Plate_${PLATE}"

mkdir -p "$OUTDIR"

echo "===================================="
echo "PLATE: $PLATE"
echo "START: $(date)"
echo "R1: $R1"
echo "R2: $R2"
echo "OUT: $OUTDIR"
echo "===================================="

# ---------------- SAFETY CHECKS ----------------
echo "Checking FASTQ integrity..."

gzip -t "$R1" || { echo "❌ CORRUPT R1: $R1"; exit 1; }
gzip -t "$R2" || { echo "❌ CORRUPT R2: $R2"; exit 1; }

echo "FASTQ files OK"

# ---------------- RUN RADTAGS ----------------
process_radtags \
  -1 "$R1" \
  -2 "$R2" \
  -b "$BC" \
  -o "$OUTDIR" \
  -e sbfI \
  -r -c -q \
  --inline_null \
  2>&1 | tee "logs/NV_C1702_${PLATE}.log"

echo "DONE: Plate $PLATE at $(date)"
```
#### Steps 3 b. Demultiplex reads using the process_radtag command in STACKS for land 4_6
```bash
#!/bin/bash
#SBATCH -p batch
#SBATCH --job-name=4_6_Demux
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=13
#SBATCH --time=70:00:00
#SBATCH --array=0-2%1
#SBATCH --mail-type=BEGIN,END,FAIL
#SBATCH --mail-user=bistbs@miamioh.edu
#SBATCH -o logs/demux_%A_%a.out
#SBATCH -e logs/demux_%A_%a.err

set -euo pipefail

module load stacks-2.66 2>/dev/null

DATA_DIR="/shared/jezkovt_bistbs_shared/Brook_trout_Project/Lane_4_6/NV-C1703"
OUT_BASE="/shared/jezkovt_bistbs_shared/Brook_trout_Project/Lane_4_6/Processed_Radtags_4_6"

PLATES=("A" "B" "C")

PLATE=${PLATES[$SLURM_ARRAY_TASK_ID]}

R1="${DATA_DIR}/NV_C1703${PLATE}_CKDL260007506-1A_23K3H2LT3_L6_1.fq.gz"
R2="${DATA_DIR}/NV_C1703${PLATE}_CKDL260007506-1A_23K3H2LT3_L6_2.fq.gz"
BC="${DATA_DIR}/C1703${PLATE}_barcodes.tsv"

OUTDIR="${OUT_BASE}/Parsed_Plate_${PLATE}"

mkdir -p "$OUTDIR"
mkdir -p logs

echo "===================================="
echo "Plate: $PLATE"
echo "Start: $(date)"
echo "===================================="

# ---- SAFETY CHECK (VERY IMPORTANT) ----
echo "Checking FASTQ integrity..."

gunzip -t "$R1" || { echo "❌ CORRUPT: $R1"; exit 1; }
gunzip -t "$R2" || { echo "❌ CORRUPT: $R2"; exit 1; }

echo "FASTQ OK"

# ---- RUN PROCESS RADTAGS ----
process_radtags \
  -1 "$R1" \
  -2 "$R2" \
  -b "$BC" \
  -o "$OUTDIR" \
  -e sbfI \
  -r -c -q \
  --inline_null \
  2>&1 | tee "logs/NV_C1703_${PLATE}.log"

echo "Finished Plate ${PLATE} at $(date)"
```

#### Step 4. Check the size of each demultiplexed samples.
- Find all of the files greater than a specific size and move only those files that are greater into another new folder.
- Only take samples with size greater than 1M.
```bash
ls -sh
find .  -size +1M -exec mv {} ../Brook_trout_Demultiplexed/ \;
```
#### Step 5. Right now it is . 1 and .2. Now, we want to rename with .R1 and .R2. 
- Doing this for ipyrad
  ```bash
rename .1. _R1_. *
rename .2. _R2_. *
```

#### Step 6. Activate the ipyrad
- First of all, create the personal ipyrad environment.
# To run Ipyrad you need to create your own personal ipyrad environment 
```bash
module load anaconda-python3
conda create -n my-ipyrad
conda update -n base -c defaults conda      #update the latest version of conda
conda activate my-ipyrad
conda install ipyrad -c conda-forge -c bioconda
```
#### Step 6 a. Create the paramter file for running the ipyrad for aligning with the ref. genome assembly.
- Ipyrad consists of seven steps (https://ipyrad.readthedocs.io/en/master/).
- First we need to create a new parameter file for the assembly.
- I created a params-brook-trout.txt file for setting the thresholds and values.

```bash
------- ipyrad params file (v.0.9.102)------------------------------------------
Brook_trout                     ## [0] [assembly_name]: Assembly name. Used to name output directories for assembly steps
/home/bistbs/Brook_trout_ipyrad/Demultiplexed_Samples_1_6 ## [1] [project_dir]: Project dir (made in curdir if not present)
                               ## [2] [raw_fastq_path]: Location of raw non-demultiplexed fastq files
                               ## [3] [barcodes_path]: Location of barcodes file
*fq.gz                               ## [4] [sorted_fastq_path]: Location of demultiplexed/sorted fastq files
reference                         ## [5] [assembly_method]: Assembly method (denovo, reference)
/home/bistbs/Brook_trout_ipyrad/ipyrad/GCF_029448725.1_ASM2944872v1_genomic.fna  ## [6] [reference_sequence]: Location of reference sequence file
rad                      ## [7] [datatype]: Datatype (see docs): rad, gbs, ddrad, etc.
TGCAGG                 ## [8] [restriction_overhang]: Restriction overhang (cut1,) or (cut1, cut2)
5                              ## [9] [max_low_qual_bases]: Max low quality base calls (Q<20) in a read
33                             ## [10] [phred_Qscore_offset]: phred Q score offset (33 is default and very standard)
6                              ## [11] [mindepth_statistical]: Min depth for statistical base calling
6                              ## [12] [mindepth_majrule]: Min depth for majority-rule base calling
10000                          ## [13] [maxdepth]: Max cluster depth within samples
0.85                           ## [14] [clust_threshold]: Clustering threshold for de novo assembly
0                              ## [15] [max_barcode_mismatch]: Max number of allowable mismatches in barcodes
2                              ## [16] [filter_adapters]: Filter for adapters/primers (1 or 2=stricter)
35                             ## [17] [filter_min_trim_len]: Min length of reads after adapter trim
2                              ## [18] [max_alleles_consens]: Max alleles per site in consensus sequences
0.05                           ## [19] [max_Ns_consens]: Max N's (uncalled bases) in consensus
0.05                           ## [20] [max_Hs_consens]: Max Hs (heterozygotes) in consensus
10                              ## [21] [min_samples_locus]: Min # samples per locus for output
0.2                            ## [22] [max_SNPs_locus]: Max # SNPs per locus
8                              ## [23] [max_Indels_locus]: Max # of indels per locus
0.5                            ## [24] [max_shared_Hs_locus]: Max # heterozygous sites per locus
0, 0, 0, 0                     ## [25] [trim_reads]: Trim raw read edges (R1>, <R1, R2>, <R2) (see docs)
0, 0, 0, 0                     ## [26] [trim_loci]: Trim locus edges (see docs) (R1>, <R1, R2>, <R2)
*                        ## [27] [output_formats]: Output formats (see docs)
                               ## [28] [pop_assign_file]: Path to population assignment file
                               ## [29] [reference_as_filter]: Reads mapped to this reference are removed in step 3

```
#### Step 6 b. Create the paramter file for running the ipyrad for aligning with the ref. genome assembly.
- Run the batch script for ipyrad
```bash
#!/bin/bash -l
#SBATCH --account=bio260092
#SBATCH --job-name=ipyrad_BT
#SBATCH --partition=shared
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=1
#SBATCH --cpus-per-task=40
#SBATCH --time=48:00:00
#SBATCH --mem=128G
#SBATCH --output=ipyrad_%j.out
#SBATCH --error=ipyrad_%j.err

# Prevent Python from buffering output so you see it in the .out file live
export PYTHONUNBUFFERED=1

cd /anvil/scratch/x-bbist/ipyrad_34567

# Load anaconda and activate your environment
module load anaconda
source $(conda info --base)/etc/profile.d/conda.sh
conda activate ipyrad_new

# Run ipyrad using the exact Python path you confirmed
# We use 'python -m ipyrad' to ensure the module is called correctly
/home/x-bbist/.conda/envs/2024.02-py311/ipyrad_new/bin/python -m ipyrad -p params-Brook_trout.txt -s 7 -c 4 -f
```
#### Step . Variant filtration using vcf-tools
```bash
# STEP 1: Mask low-depth genotypes and calculate missingness
vcftools \
  --vcf /anvil/scratch/x-bbist/Trout_Variant_Filtration/Brook_trout.vcf \
  --minDP 5 \
  --missing-indv \
  --out /anvil/scratch/x-bbist/Trout_Variant_Filtration/Brook_trout

# STEP 2: Create list of individuals with >60% missing data
awk '$5 > 0.60 {print $1}' \
/anvil/scratch/x-bbist/Trout_Variant_Filtration/Brook_trout.imiss \
> /anvil/scratch/x-bbist/Trout_Variant_Filtration/lowDP.indv

# STEP 3: Apply all final filters simultaneously
vcftools \
  --vcf /anvil/scratch/x-bbist/Trout_Variant_Filtration/Brook_trout.vcf \
  --remove /anvil/scratch/x-bbist/Trout_Variant_Filtration/lowDP.indv \
  --max-missing 0.60 \
  --maf 0.05 \
  --minDP 5 \
  --min-meanDP 5 \
  --min-alleles 2 \
  --max-alleles 2 \
  --remove-indels \
  --thin 10000 \
  --recode \
  --recode-INFO-all \
  --out /anvil/scratch/x-bbist/Trout_Variant_Filtration/Brook_trout.filtered.biallelic

```

#### Finding out the events:
#### Step 1: Convert VCF to Haplotype format
```bash
./RADpainter hapsFromVCF ../../Brook_trout.filtered.biallelic.recode.vcf > trout_haps.txt
```
#### Step 2: Calculate the Co-ancestry Matrix (Painting)
- It calculates how much DNA is shared between every individual fish.
```bash
#!/bin/bash
#SBATCH -J RADpainter_Trout
#SBATCH -o %x.%j.out
#SBATCH -e %x.%j.err
#SBATCH -p shared
#SBATCH -n 1
#SBATCH -c 16
#SBATCH -t 04:00:00
#SBATCH --mem=32G

# 1. Load the environment you already set up
module load gsl gcc openmpi

# 2. Set your working directory paths
BASE_DIR="/anvil/scratch/x-bbist/Trout_Variant_Filtration/FineStructure"
SOFTWARE_DIR="${BASE_DIR}/fineRADstructure"
OUTPUT_DIR="${BASE_DIR}/Output"

# 3. Move into the software directory to run the binary
cd $SOFTWARE_DIR

# 4. Run Step 2 (Painting)
# We use the full path for the input and output to ensure they land in /Output
echo "Starting RADpainter paint at $(date)"

./RADpainter paint trout_haps.txt

# 5. Move the resulting files to your Output folder
# RADpainter creates [input_name]_chunks.out and [input_name]_indivs.txt
mv trout_haps_chunks.out ${OUTPUT_DIR}/
mv trout_haps_indivs.txt ${OUTPUT_DIR}/

echo "Finished RADpainter paint at $(date)"
```
#### Step 3: Run the Clustering (MCMC)
- Now we assign individuals to groups. We use 100,000 iterations for a robust result.
```bash
#!/bin/bash
#SBATCH -J fineStructure_MCMC
#SBATCH -o /anvil/scratch/x-bbist/Trout_Variant_Filtration/FineStructure/Output/MCMC_Trout/%x.%j.out
#SBATCH -e /anvil/scratch/x-bbist/Trout_Variant_Filtration/FineStructure/Output/MCMC_Trout/%x.%j.err
#SBATCH -p shared
#SBATCH -n 1
#SBATCH -c 40
#SBATCH -t 70:00:00
#SBATCH --mem=32G

# 1. Load the environment
module load gsl gcc openmpi

# 2. Define the Target Directory
TARGET_DIR="/anvil/scratch/x-bbist/Trout_Variant_Filtration/FineStructure/Output/MCMC_Trout"
BIN_DIR="/anvil/scratch/x-bbist/Trout_Variant_Filtration/FineStructure/fineRADstructure"
INPUT_CHUNKS="/anvil/scratch/x-bbist/Trout_Variant_Filtration/FineStructure/Output/trout_haps_chunks.out"

# Create the directory if it doesn't exist yet
mkdir -p $TARGET_DIR

# 3. Move into the target directory so all output stays there
cd $TARGET_DIR

# 4. Run Step 3 (MCMC Clustering)
echo "Starting fineStructure MCMC at $(date)"

$BIN_DIR/finestructure -x 100000 -y 100000 -z 1000 $INPUT_CHUNKS trout_mcmc.xml

echo "MCMC finished at $(date)"

# 5. Run Step 4 (Tree Building)
echo "Starting Tree Building at $(date)"

$BIN_DIR/finestructure -m T -x 10000 $INPUT_CHUNKS trout_mcmc.xml trout_tree.xml

echo "Tree Building finished at $(date)"
```

#### Step 4: Build the Tree
- Finally, build the phylogenetic tree that shows how these identified clusters relate to one another.
```bash
./finestructure -m T -x 10000 trout_haps_chunks.out trout_mcmc.xml trout_tree.xml
```

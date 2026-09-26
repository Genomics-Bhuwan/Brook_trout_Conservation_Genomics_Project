#### Data Summary
- **Organism:** Brook Trout (*Salvelinus fontinalis*)
- **Sample Size:** 570 individuals across 18 populations
- **Initial Variants:** 30,364 biallelic SNPs
- **Retained Variants (LD-pruned):** 2,849 unlinked SNPs
#### Running BayesAss for inferrring the current gene flow for last 1-3 generations:
- Link: https://github.com/brannala/BA3/wiki
#### Contemporary Migration Analysis of Brook Trout (*Salvelinus fontinalis*) using BayesAss 3.5.0
- This directory contains the pipeline for LD-pruning high-density RADseq/SNP data and estimating contemporary migration rates across 18 Brook Trout populations using BayesAss 3.5.0 (BA3)

#### Pipeline & Methods Workflow
- Step 1: Linkage Disequilibrium (LD) Pruning
High SNP density creates sharp posterior distributions that cause MCMC proposal step sizes to hit extreme limits ($dM = 0.990$). To prevent this, we pruned linked loci using PLINK 1.9 with a strict pairwise threshold ($r^2 < 0.02$).

```bash
# Convert raw VCF to PLINK binary format
./plink --vcf Brook_trout.filtered.biallelic.recode.vcf \
  --allow-extra-chr \
  --make-bed \
  --out bt_plink

# Perform LD pruning (Window: 50 SNPs, Step: 5 SNPs, r^2 threshold: 0.02)
./plink --bfile bt_plink \
  --allow-extra-chr \
  --indep-pairwise 50 5 0.02 \
  --out bt_ld_strict

# Extract unlinked loci back to VCF format using VCFtools
vcftools --vcf Brook_trout.filtered.biallelic.recode.vcf \
  --snps bt_ld_strict.prune.in \
  --recode --recode-INFO-all \
  --out Brook_trout_LD_strict

```

- Step 2. Running BA3 program 

```bash
# Absolute paths
BA3_EXEC="/home/bistbs/Brook_trout_ipyrad/BA3/BA3"
VCF_FILE="/home/bistbs/Brook_trout_ipyrad/BA3/Linkage_Disequilibrium_Method/Brook_trout_LD_strict.recode.vcf"
POPMAP_FILE="/home/bistbs/Brook_trout_ipyrad/BA3/Linkage_Disequilibrium_Method/popmap.txt"
SEEDS=(12345 23456 34567 45678 56789)

TOTAL_RUNS=${#SEEDS[@]}
CURRENT_RUN=1

for SEED in "${SEEDS[@]}"; do
    DIR="/home/bistbs/Brook_trout_ipyrad/BA3/Linkage_Disequilibrium_Method/run_seed_${SEED}"
    
    echo "============================================================"
    echo " Starting BayesAss Run [${CURRENT_RUN}/${TOTAL_RUNS}] | Seed: ${SEED}"
    echo " Directory: ${DIR}"
    echo "============================================================"
    
    # Create and enter seed directory
    mkdir -p "${DIR}"
    cd "${DIR}" || exit 1

    # Execute BA3 with proper option ordering
    "${BA3_EXEC}" \
      -i 50000000 \
      -b 5000000 \
      -n 100 \
      -m 1.00 -a 1.00 -f 1.00 \
      -s "${SEED}" \
      -t -u -v \
      -V "${VCF_FILE}" \
      -M "${POPMAP_FILE}" \
      -o "${DIR}/BT_BA3_seed${SEED}_50M.out" \
      "${VCF_FILE}"

    # Move generated trace and indiv files into seed directory
    if [ -f "pop.str.trace.txt" ]; then
        mv pop.str.trace.txt "${DIR}/BT_BA3_seed${SEED}.trace.txt"
    fi
    if [ -f "pop.str.indiv.txt" ]; then
        mv pop.str.indiv.txt "${DIR}/BT_BA3_seed${SEED}.indiv.txt"
    fi

    # Return to main working directory
    cd /home/bistbs/Brook_trout_ipyrad/BA3/Linkage_Disequilibrium_Method

    echo ""
    echo "Finished run for Seed: ${SEED}"
    echo ""
    
    CURRENT_RUN=$((CURRENT_RUN + 1))
done

echo "All 5 BayesAss runs completed!"
```

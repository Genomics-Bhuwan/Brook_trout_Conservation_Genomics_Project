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
/home/bistbs/Brook_trout_ipyrad/BA3/BA3 -c \
  -V Brook_trout_LD_strict.recode.vcf \
  -M /home/bistbs/Brook_trout_ipyrad/BA3/popmap.txt \
  -o BA3_LD_fixed_out.txt \
  -i 10000000 -b 1000000 -n 1000 -t -g \
  -m 0.15 -a 0.10 -f 0.10 -N -s 42 \
  -F allele_freqs_LD_fixed.tsv
```

```bash
#!/bin/bash

VCF="/home/bistbs/Brook_trout_ipyrad/Downstream_Analysis_Brook_trout/Brook_trout.filtered.biallelic.recode.vcf"
POP_DIR="subpops_list"
OUT_DIR="fst_results"

mkdir -p ${OUT_DIR}

# Get list of subpopulation names
POPS=($(ls ${POP_DIR}/*.txt | sed 's/subpops_list\///g' | sed 's/\.txt//g'))
NUM_POPS=${#POPS[@]}

# Output pairwise results log
RESULT_FILE="${OUT_DIR}/pairwise_fst_summary.txt"
echo -e "Pop1\tPop2\tWeighted_Fst" > ${RESULT_FILE}

echo "Starting pairwise Fst calculations across ${NUM_POPS} subpopulations..."

# Loop over all population pairs
for (( i=0; i<${NUM_POPS}; i++ )); do
  for (( j=i+1; j<${NUM_POPS}; j++ )); do
    P1=${POPS[$i]}
    P2=${POPS[$j]}
    
    OUT_PREFIX="${OUT_DIR}/${P1}_vs_${P2}"
    
    # Run VCFtools pairwise Fst
    vcftools --vcf ${VCF} \
             --weir-fst-pop ${POP_DIR}/${P1}.txt \
             --weir-fst-pop ${POP_DIR}/${P2}.txt \
             --out ${OUT_PREFIX} &> ${OUT_PREFIX}.log
             
    # Extract weighted Fst from the log output
    FST_VAL=$(grep "Weir and Cockerham weighted Fst estimate:" ${OUT_PREFIX}.log | awk '{print $NF}')
    
    echo -e "${P1}\t${P2}\t${FST_VAL}" >> ${RESULT_FILE}
  done
done

echo "Pairwise Fst calculations completed!"
```

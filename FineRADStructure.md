#### Finding out the events:
#### Step 1: Convert VCF to Haplotype format
```bash
./RADpainter hapsFromVCF -i ../../Brook_trout.filtered.biallelic.recode.vcf -o trout_haps.txt
```
#### Step 2: Calculate the Co-ancestry Matrix (Painting)
- It calculates how much DNA is shared between every individual fish.
```bash
./RADpainter paint trout_haps.txt
```
#### Step 3: Run the Clustering (MCMC)
- Now we assign individuals to groups. We use 100,000 iterations for a robust result.
```bash
./finestructure -x 100000 -y 100000 -z 1000 trout_haps_chunks.out trout_mcmc.xml
```

#### Step 4: Build the Tree
- Finally, build the phylogenetic tree that shows how these identified clusters relate to one another.
```bash
./finestructure -m T -x 10000 trout_haps_chunks.out trout_mcmc.xml trout_tree.xml
```

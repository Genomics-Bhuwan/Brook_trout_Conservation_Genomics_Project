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

#### Step 2. a. Clone filtering for Lane 4-6
#### Make a directory to output files into
```bash
#!/bin/bash
#SBATCH -A bio260092                # Changed from x-bio260092 to bio260092
#SBATCH -p shared
#SBATCH --job-name=4_6clone_filter
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=13          # 8 CPUs is plenty for clone_filter
#SBATCH --time=46:00:00             # 24 hours is safer for the shared partition
#SBATCH --array=0-2
#SBATCH -o logs/cf_%A_%a.out
#SBATCH -e logs/cf_%A_%a.err

# Load the compiler environment
module load gcc

# 1. Path to your compiled bin folder
STACKS_BIN="/anvil/projects/x-bio260092/Brook_Trout_Project/Lane_4_6/NV-C1703/Clone_Filtered_Lane_4_6/stacks-2.68/bin"

# 2. Sample List
SAMPLES=("NV_C1703A" "NV_C1703B" "NV_C1703C")
SAMPLE=${SAMPLES[$SLURM_ARRAY_TASK_ID]}

# 3. Directory Setup
DATA_DIR="/anvil/projects/x-bio260092/Brook_Trout_Project/Lane_4_6/NV-C1703"
OUT_DIR="/anvil/projects/x-bio260092/Brook_Trout_Project/Lane_4_6/NV-C1703/Clone_Filtered_Lane_4_6"

mkdir -p $OUT_DIR
mkdir -p logs

echo "Starting clone_filter for $SAMPLE..."

# 4. Run the command
${STACKS_BIN}/clone_filter -1 ${DATA_DIR}/${SAMPLE}_*_1.fq.gz \
                           -2 ${DATA_DIR}/${SAMPLE}_*_2.fq.gz \
                           -i gzfastq \
                           -o $OUT_DIR \
                           --inline_null \
                           --oligo_len_1 8

echo "Finished $SAMPLE at $(date)"
```

#### Step 2. b. Clone filtering for Lane 1-3
# Load the module
```bash
#!/bin/bash
#SBATCH -A bio260092
#SBATCH -p shared
#SBATCH --job-name=clone_filter_1_3
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=13
#SBATCH --time=46:00:00
#SBATCH --array=0-2
#SBATCH -o logs/cf_%A_%a.out
#SBATCH -e logs/cf_%A_%a.err

module load gcc

# Path to stacks binaries
STACKS_BIN="/anvil/projects/x-bio260092/Brook_Trout_Project/Lane_4_6/NV-C1703/Clone_Filtered_Lane_4_6/stacks-2.68/bin"

# Samples (match your filenames exactly)
SAMPLES=("NV_C1702A" "NV_C1702B" "NV_C1702C")
SAMPLE=${SAMPLES[$SLURM_ARRAY_TASK_ID]}

# Input + output directories
DATA_DIR="/anvil/projects/x-bio260092/Brook_Trout_Project/Lane_1_3/NV-C1702"
OUT_DIR="/anvil/projects/x-bio260092/Brook_Trout_Project/Lane_1_3/Clone_Filtering_Lane_1_3"

mkdir -p "$OUT_DIR"
mkdir -p logs

echo "Starting clone_filter for $SAMPLE..."

${STACKS_BIN}/clone_filter \
    -1 ${DATA_DIR}/${SAMPLE}_*_1.fq.gz \
    -2 ${DATA_DIR}/${SAMPLE}_*_2.fq.gz \
    -i gzfastq \
    -o $OUT_DIR \
    --inline_null \
    --oligo_len_1 8

echo "Finished $SAMPLE at $(date)"


```   

# Check to see if the barcode is on the end or not, if not, then we need to use trimmomatic to make sure they are
# Remember for inline_index they are only going to be on the R1 read   

Ls #List all files and check if the filtered files do possess barcodes  in the end or not.
zcat Dasy1_S1_L001_R1_001.1.fq.gz | head

#######################
##### Demultiplexing #####
#######################
# Demultiplex reads using the process_radtag command in STACKS
#The process_radtags command in STACKS is used for demultiplexing, quality filtering, and trimming of reads generated from RAD-seq or related experiments. It allows you to sort and separate reads by individual sample based on barcode/index sequences and perform initial quality control.
# This can be done individually file by file or by writing a shell script (preferred method for time efficiency)
# First we write a loop to create a directory to hold each pool of individuals
# This code says that for number 1 to 6 we make a directory names Dasy_1, Dasy_2, etc
# The Dasy_undet will be for the undetermined file
# The semi colon between each command allows us to run the for loop on one line but tells the computer to treat the commands as though they are on separate lines
# Your upper limit should be how many pools you have, for example, if you have 8 the 6 would be an 8

for i in {1..6}; 
do mkdir Parsed_Dasy_$i; 
mkdir Parsed_Dasy_undet; done

OR
## Example of how to demultiplex one pool at a time
process_radtags -1  ./Dasy1_S1_L001_R1_001.1.fq.gz -2 ./Dasy1_S1_L001_R2_001.2.fq.gz -b /home/bistbs/Raw_Sequence_Data/Lane1_Seq_Jan2023/Dasypus_barcodes.txt -o ./Parsed_Dasy_1 -r --inline_index --renz_1 sbfI  --renz_2 sau3AI --barcode_dist_2 2  --disable_rad_check

## Or you can write a for loop to do it semi-programatically

# Now we write the for loop to create the stacks demultiplexing commands
# First we echo the module that we need (stacks-2.55)

echo module load stacks-2.66> Dasypus_novemcinctus_process_radtags.sh

# Write the for loop, you will need to modify the sed command and the subsequent file names for -1 and -2 to match the L00 number on your files. Here my files were named Dasy1_S1_L001_R1_001.1.fq.gz but they could have been Dasy1_S1_L002_R1_002.1.fq.gz, then I need to change the sed command to sed 's/\_R1_002.*.fq.gz//' and the -1 and -2 to ${i}_R1_002.1.fq.gz
# Finally you will need to edit the sh file to match the output directory (-o flag)

for i in `ls -1 *.1.fq.gz | sed 's/\_R1_001.*.fq.gz//'`; do echo process_radtags -1 ./${i}_R1_001.1.fq.gz -2 ./${i}_R2_001.2.fq.gz -b /shared/jezkovt_bistbs_shared/Data/Raw_Sequence_Data/Lane1_Seq_Jan2023/Clone_Filtered/Dasypus_barcodes.txt -o ./Parsed_Dasy_ -r --inline_index --renz_1 sbfI --renz_2 sau3AI --barcode_dist_2 2  --disable_rad_check >> Dasypus_novemcinctus_process_radtags.sh; done

(Some time there are whitespaces or gaps due to which there is error). Remove the gaps from the barcode files.

#Once you ran the for loop from the up command. Check the “Dasypus_novemcinctus_process_radtags.sh” folder in activities in nomachine and put _1 or _2 or _3 or _4 or _5 or _6 after ./Parsed_Dasy_. And save this one by going to the File menu and save.



#Description of the above code
# list all the files with the extension .1.fg.gz and send the output to the sed for additional processing.
#sed performs the substitution within the output received and aims to replace the matched pattern with an empty string. S indicated substitution with sed. 
# /\ indicated matches pattern should be replaced by an empty string effectively removing the matches part from the output.
# and do echo process_radtags -1 ./${i}_R1_001.1.fq.gz -2 ./${i}_R2_001.2.fq.gz -b /shared/jezkovt_shared/Dasypus_RADseq/Dasypus_barcodes.txt echoes the process_radtags to display the command process_radtags and other field shows path to the first and second input file. And also shows the path to the barcoded file. 
#output file will be the Parsed_Dasy_ where the processed files will be stored.
# and -r denotes the input file is paired end format. 
#--inline-index- barcode is present in the sequence header rather than a separate file. 

# Get a job
/usr/bin/salloc -N 1 -n 8 -c 2 -J interactive -p batch -t 72:00:00  /usr/bin/srun  --interactive --pty /bin/bash

sh Dasypus_novemcinctus_process_radtags.sh    [This takes long long time to run as many files are being written in shell script ask for ] 

#above command executes the shell script using the sh interpreter. 
#Dasypus_novemcinctus_process_radtags.sh: This is the name of the shell script file that contains a series of process_radtags commands or other shell commands related to processing RAD-seq data.

# Make a directory for all of the demultiplexed samples
mkdir ./D_novemcinctus_demultiplexed_samples

# Move into a directory of demultiplexed files
cd ./Parsed_Dasy_1

# Inspect the results, you should expect at least 95% of your reads to be retained
# Check your process rad tags call and previous steps if you don’t get 95% retention
cat process_radtags.Clone_Filtered.log 

# Look at all of the files in the directory and their size
ls -sh
# Find all of the files greater than a specific size and move into the demultiplexed directory
find .  -size +1M -exec mv {} ../D_novemcinctus_demultiplexed_samples/ \;

# Change back into the demultiplexed directory
cd ../D_novemcinctus_demultiplexed_samples/

# Count to make sure that you have the correct number of files
# If its PE you should have the number of samples*2, for example, if I have 56 samples I expect there to be 112 files

ls | wc -l

Run the above command for respective cd ./Parsed_Dasy_2, cd ./Parsed_Dasy_3, cd ./Parsed_Dasy_4, cd ./Parsed_Dasy_5, cd ./Parsed_Dasy_6  upto the steps : ls | wc-l 


#For Parsed_Dasy_2
cd ./Parsed_Dasy_2
cat process_radtags.Clone_Filtered.log 
ls -sh
find .  -size +1M -exec mv {} ../D_novemcinctus_demultiplexed_samples/ \;
cd ../D_novemcinctus_demultiplexed_samples/
ls | wc -l

#For Parsed_Dasy_3
cd ./Parsed_Dasy_3
cat process_radtags.Clone_Filtered.log 
ls -sh
find .  -size +40M -exec mv {} ../D_novemcinctus_demultiplexed_samples/ \;
cd ../D_novemcinctus_demultiplexed_samples/
ls | wc -l

#For Parsed_Dasy_4
cd ./Parsed_Dasy_4
cat process_radtags.Clone_Filtered.log 
ls -sh
find .  -size +10M -exec mv {} ../D_novemcinctus_demultiplexed_samples/ \;
cd ../D_novemcinctus_demultiplexed_samples/
ls | wc -l


#For Parsed_Dasy_5
cd ./Parsed_Dasy_5
cat process_radtags.Clone_Filtered.log 
ls -sh
find .  -size +10M -exec mv {} ../D_novemcinctus_demultiplexed_samples/ \;
cd ../D_novemcinctus_demultiplexed_samples/
ls | wc -l


#For Parsed_Dasy_6
cd ./Parsed_Dasy_6
cat process_radtags.Clone_Filtered.log 
ls -sh
find .  -size +1M -exec mv {} ../D_novemcinctus_demultiplexed_samples/ \;
cd ../D_novemcinctus_demultiplexed_samples/
ls | wc -l






######## Rename files for ipyrad#####
rename .1. _R1_. *
rename .2. _R2_. *


############################
##### Assemble with Ipyrad #####
############################
# To run Ipyrad you need to create your own personal ipyrad environment 
# This only needs to be done once
# You should probably name you ipyrad environment something useful such as BSB-ipyrad instead of my-ipyrad 
On the head node:
module load anaconda-python3
conda create -n my-ipyrad
conda update -n base -c defaults conda      #update the latest version of conda
conda activate my-ipyrad
conda install ipyrad -c conda-forge -c bioconda

Then, allocate a computer node:
/usr/bin/salloc -N 1 -n 8 -c 2 -J interactive -p batch -t 240:00:00  /usr/bin/srun  --interactive --pty /bin/bash
conda activate my-ipyrad
conda init bash
then:
log out of interactive session and allocate a new job,
/usr/bin/salloc -N 1 -n 8 -c 2 -J interactive -p batch -t 240:00:00  /usr/bin/srun  --interactive --pty /bin/bash

then again (the "conda init bash" looks like it's a one time procedure):
conda activate my-ipyrad

### Load Ipyrad 
module load anaconda-python3
source /software/python/anaconda3/etc/profile.d/conda.sh
conda activate kf-ipyrad # this line changes based on what you named your environment

# Ipyrad consists of seven steps (https://ipyrad.readthedocs.io/en/master/)
# First we need to create a new parameter file for the assembly

ipyrad -n armadillos
# Let’s open the file and see what we’ve got 

nano params-armadillos.txt 

 



# We will edit the parameter file for our data and the general settings that we want to use in the assembly 
# Since we already have demultiplexed files we will edit parameter 4 and specify that all of our files are .fq.gz (*.fq.gz) # /home/bistbs/Raw_Sequence_Data/Lane1_Seq_Jan2023/Clone_Filtered/D_novemcinctus_demultiplexed_samples/. This should be the location of the demultiplex files. 

# We also need to edit parameter 7 so that it is pairddrad, this indicates that we used ddradseq and sequenced the samples pair-end, set it to ddrad if you are using single-end data (or just R1s)
# Next we edit parameter 8 to reflect the overhang for our restriction enzymes
# SbfI = TGCAGG, Sau3AI = GATC
# You can check these on neb’s website (hyperlink above)
# Another way to check you sequences is to use the command zcat myseq_R1_.fq.gz | head
# Then you look for the invariable sequence at the beginning of each read
# This is also a good time to make sure that you do not have any barcodes leftover in your data (trim them if you do)
# Now we edit parameter 14 and make it 0.9. This is a good setting for related species, but you may need to relax it to 0.85 if you are working with distant species.
# Next we make sure that parameter 16 = 2. This will filter out any illumina adapters present in our data
# Then we edit parameter 21 to half of our number of samples or higher (your call). 50% is a good starting point and you can branch the assembly later if you want. 
# Finally, we edit parameter 27 and change it to * so that it outputs all of the possible formats. 




### Run Ipyrad step by step
# We use the ipyrad command, specify the parameter file with -p and the step that we want to perform with -s

# Run step 1
ipyrad -p params-armadillos.txt -s 1 
# Run step 2
ipyrad -p params-armadillos.txt -s 2

# Run the remaining steps 
ipyrad -p params-armadillos.txt -s 34567

## Or  run it all at once 
ipyrad -p params-armadillos.txt -s 1234567




##vcf filtering steps and then popgenHelpR



#######Sending the POOLS to the Sequencing Centre
Purpose: To submit ddRADseq sequencing request to the University of Oregon GC3F
Author: Keaka Farleigh (farleik@miamioh.edu; keakafarleigh@gmail.com)
Date: January 27th, 2023

Notes: Submit different requests for each lane that is being sequenced, even if they are the same species. 

Example Barcode Sheet Format
Sample	Individual Barcode	Pool Barcode	Pool
Sample1	CCCATA		TGACCA 		1
Sample2	CGAAAC      		ACAGTG 		2

Workflow
There are two parts to submitting a sequencing request. First, we submit a sequencing request and then we submit a sample prep request. 

Steps
Log into iLab (https://uoregon.ilab.agilent.com/account/login)
Click request services and select your sequencing platform. 
As of 1-27-2023, this is the Novaseq 6000, but that may change.

Fill in the general information section, including the name of your sample, library type, and date that you will ship it.


Fill in your run configuration. Do not blindly copy the picture below, make sure you talk to Tereza to decide what you want to do.

Fill in the library details information. Again, do not blindly copy this information from the picture below..

Fill in the primers and indexing section. Our barcode kit is listed as other, but we note that they are Illumina compatible. Make sure to say Yes that index reads are required and to request demultiplexing. Our samples are dual-indexed, 6 nucleotides long.


Fill in the miscellaneous and additional comments section. I usually upload the barcodes as an excel sheet and leave a note thanking them and with my and Tereza’s email. 

9. Scroll to the bottom of the page, make sure that your cost and payment information is correct and submit the request to the core.
10. Move onto the sample prep services request. 

11. Click request service and fill in the information. Make sure to list that you are requesting QC/multiplexing & size selection/concentration. The number of samples is equal to the number of pools that you have.

12. Upload your barcodes file as the sample spreadsheet.
13. Submit the request to the core. 
14. You can check the status of your request in iLab or by looking at the Illumina sequencing queue on the GC3F website (https://gc3f.uoregon.edu/illumina-sequencing/job/listReceived).



# Assemble nuclear rDNA (18S and 28S) from genomic data using BBmap and MITObim
This workflow is in two parts. The first part will convert trimmed forward and reverse reads into a single interleaved file uisng BBmap. The second part will use the interleavd data and a user provided seed to assemble nuclear ribosomal sequences using MITObim.

## Part 1
## Convert trimmed reads into interleaved format using BBmap
### File preparation
1. Create a text file called "bbmap_samples.txt" with each sample ID in a column. Use unique sample IDs.

```
SAMPLE_ID_1
SAMPLE_ID_2
SAMPLE_ID_3
```

2. The trimmed reads file names for each sample should have the sample IDs from step 1 followed by "_R1_trimmed.fastq.gz" and "_R2_trimmed.fastq.gz" for the forward and reverse reads (R1 and R2).

### Run BBmap on Hydra 
After preparing the input files (steps 1 and 2 above), run the job below on Hydra in the same directory as the trimmed reads files and the "bbmap_samples.txt" file

When complete, the interleaved sequence files will be in a directory called "interleaved_sequences"

```
# /bin/sh
# ----------------Parameters---------------------- #
#$ -S /bin/sh
#$ -pe mthread 3
#$ -q sThC.q
#$ -l mres=24G,h_data=8G,h_vmem=8G
#$ -cwd
#$ -j y
#$ -N BBMap
#$ -o BBMap.log
#
# ----------------Modules------------------------- #
module load bioinformatics/bbmap
#
# ----------------Your Commands------------------- #
#
echo + `date` job $JOB_NAME started in $QUEUE with jobID=$JOB_ID on $HOSTNAME
#
mkdir -p interleaved_sequences
exec &> BBMap_log.txt
while IFS= read -r SAMPLE || [ -n "$SAMPLE" ]; do
    reformat.sh \
    in1="${SAMPLE}_R1_PE_trimmed.fastq.gz" \
    in2="${SAMPLE}_R2_PE_trimmed.fastq.gz" \
    out="./interleaved_sequences/${SAMPLE}_interleaved.fastq.gz"
done < bbmap_samples.txt
#
echo = `date` job $JOB_NAME done
```

## Part 2
## Use interleaved sequence data created in Part 1 to assemble nuclear rDNA using MITObim
### File Preparation
1. Create a text file called "mitobim_samples.txt" with each unique sample ID in a column. If the same sample IDs from Step 1 will be used, simply copy that file and rename it "mitobim_samples.txt".

2. Create a fasta file with the seed used for assembling the gene of interest. For nuclear ribosomal genes, partial 18S or 28S sequences may be used. Depending on the taxon, using either 18S or 28S MITObim may assemble the entire nuclear ribosomal operon, or may just assemble the single gene from the seed. 
 
### Run MITObim on Hydra
1. The following MITObim commands in the job file must be modified before submitting to Hydra.

-ref "name of the project"

Full paths are needed for the --readpool (reads data) --quick (seed sequence) commands.

--readpool "full path"/interleaved_sequences/${SAMPLE}_interleaved.fastq.gz 

--quick "full path to to fasta file with seed"

--end "number of iterations" 
[Note: Depending on how large you need your final contig, the number of iterations can be changed. Usually, about 4-5 iterations are needed for complete 18S or 28S. However, more or less can be used depending on the taxon and data.]

3. Run the job below in the same directory that contains the "mitobim_samples.txt" file (this path can be modified in the job file if desired). Since full paths are required for the "interleaved_sequences" directory which contains the interleaved sequences and the seed fasta file, they do not have to located in the same directory.

4. When the job completes, there should be a separate folder for each sample labeled ["sample_name"_mitobim]. The final assembled contig will be in the directory with the last assigned interation and end with "_noIUPAC.fasta". 

```
# /bin/sh
# ----------------Parameters---------------------- #
#$ -S /bin/sh
#$ -pe mthread 3
#$ -q mThC.q
#$ -l mres=18G,h_data=6G,h_vmem=6G
#$ -cwd
#$ -j y
#$ -N mitobim_loop.job
#$ -o mitobim.log
#
# ----------------Modules------------------------- #
module load ~/modulefiles/miniconda
source activate mitobim
# ----------------Your Commands------------------- #
#
echo + `date` job $JOB_NAME started in $QUEUE with jobID=$JOB_ID on $HOSTNAME
echo + NSLOTS = $NSLOTS
#
cat ./mitobim_samples.txt | while read SAMPLE
do
mkdir "${SAMPLE}_mitobim"
cd "${SAMPLE}_mitobim" || exit
MITObim.pl \
-sample "${SAMPLE}" \
-ref 28SseedCladiella \
--readpool /scratch/nmnh_corals/projects/cladiellatypes/data/trimmed_sequences/interleaved_sequences/${SAMPLE}_interleaved.fastq.gz \
--quick /scratch/nmnh_corals/projects/cladiellatypes/data/trimmed_sequences/cladiella_28S_seed.fasta \
--end 4 --pair --clean &> "log_${SAMPLE}"
cd ..
done
#
echo = `date` job $JOB_NAME done
```

## Part 3 - Extra. Pull all final fasta contings from directories and rename them with the sample IDs
### Extract final fasta files and copy them to a separate directory
1. In the shell script below, modify the path "./${mitobim_results}/iteration[DIGIT OF LAST ITERATION]/*_noIUPAC.fasta ./mitobim_final_contigs" by including a digit for "DIGIT OF LAST ITERATION". This is the directory of the last interation of MITObim that contains the final contig.

2. Run the shell script in the same directory as the [sample_mitobim] directories from the output of MITObim from step 2. Explanations of the script steps are given in the text of the script.

```
#!/bin/sh

# Creates a directory named mitobim_final_contigs
mkdir mitobim_final_contigs  

# Iterates over files/directories with names ending in _mitobim in the current directory
for mitobim_results in ./*_mitobim  

do

# Prints the name of each file/directory being processed
    echo "Processing file: $mitobim_results"  
# Copies files matching the pattern to the mitobim_final_contigs directory
    cp ./${mitobim_results}/iteration[digit of last interation]/*_noIUPAC.fasta ./mitobim_final_contigs  
    cp ./${mitobim_results}/log_* ./mitobim_final_contigs  
done
```






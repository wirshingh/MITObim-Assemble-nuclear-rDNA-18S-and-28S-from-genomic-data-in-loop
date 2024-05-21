# Assemble nuclear rDNA (18S and 28S) from genomic data using BBmap and MITObim
This workflow is in two parts. The first part will convert trimmed forward and reverse reads into a single interleaved file uisng BBmap. The second part will use the interleavd data and a user provided seed to assemble nuclear ribosomal sequences using MITObim.

## Convert trimmed reads into interleaved format using BBmap
### File preparation
1. Create a text file called "bbmap_samples.txt" with each sample name in a column.

```
SAMPLE_ID_1
SAMPLE_ID_2
SAMPLE_ID_3
```

2. The trimmed reads file names should have the sample name from step 1 followed by "_R1_trimmed.fastq.gz" and "_R2_trimmed.fastq.gz" for the forward and reverse reads (R1 and R2).

### Run BBmap on Hydra 
After preparing the input files (steps 1 and 2 above), run the job below on Hydra.

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




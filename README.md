# Assemble nuclear rDNA (18S and 28S) from genomic data using BBmap and MITObim using a loop
This workflow is in two parts. The first part will convert trimmed forward and reverse reads into a single interleaved file uisng BBmap. The second part will use the interleavd data and a user provided seed to assemble nuclear ribosomal sequences using MITObim.

## Part 1
## Convert trimmed reads into interleaved format using BBmap
Link to BBmap site - https://sourceforge.net/projects/bbmap/
### File preparation
The trimmed reads file names must end in "_R1_PE_trimmed.fastq.gz" and "_R2_PE_trimmed.fastq.gz" for the forward and reverse reads (R1 and R2) for these scripts to work. The scripts will utilize whatever text is in front of either "_R1(orR2)_PE_trimmed.fastq.gz" as the sample name. If your trimmed reads file names end with different text, the job file may also be modified accordingly. 

### Run BBmap on Hydra 
After verifying your trimmed reads file names end with the appropriate text, save the job file below as "bbmap_interleaved.job" and run the job below on Hydra (qsub bbmap_interleaved.job) in the same directory as the trimmed reads files.

When complete, the interleaved sequence files will be in a directory called "interleaved_sequences".

```
# /bin/sh
# ----------------Parameters---------------------- #
#$ -S /bin/sh
#$ -pe mthread 2
#$ -q sThC.q
#$ -l mres=6G,h_data=3G,h_vmem=3G
#$ -cwd
#$ -j y
#$ -N bbmap_interleaved
#$ -o bbmap_interleaved.log
#
# ----------------Modules------------------------- #
module load bioinformatics/bbmap
#
# ----------------Your Commands------------------- #
#
echo + `date` job $JOB_NAME started in $QUEUE with jobID=$JOB_ID on $HOSTNAME
#
mkdir -p interleaved_sequences
for GETSAMPLENAME in ./*_R1_PE_trimmed.fastq.gz
do
SAMPLENAME=$(basename "$GETSAMPLENAME" _R1_PE_trimmed.fastq.gz)
    reformat.sh \
    in1="${SAMPLENAME}_R1_PE_trimmed.fastq.gz" \
    in2="${SAMPLENAME}_R2_PE_trimmed.fastq.gz" \
    out="./interleaved_sequences/${SAMPLENAME}_interleaved.fastq.gz"
done
#
echo = `date` job $JOB_NAME done

```

## Part 2
## Use interleaved sequence data created in Part 1 to assemble nuclear rDNA using MITObim in a loop
link to MITObim - https://github.com/chrishah/MITObim
### File Preparation
Create a fasta file (.fasta) with the seed used for assembling the gene of interest, and place it the same directory as the trimmed reads files. For nuclear ribosomal genes, partial or complete 18S or 28S sequences may be used. Depending on the taxon, using either 18S or 28S, MITObim may assemble the entire nuclear ribosomal operon, or may just assemble the single gene from the seed. 
 
### Run MITObim on Hydra
#### NOTE: The following MITObim commands in the job file must be modified before submitting to Hydra.

-ref name_of_project 

change this to the name of your project.

--readpool "full path to"/interleaved_sequences/${SAMPLE}_interleaved.fastq.gz 

change "full path to" with the path to the interleaved_sequences directory.

--quick "full path to to fasta file with seed" 

paste the full path to the directory that contains the fasta file with the 18S or 28S seed.

--end 4 

#### Note: Depending on how large you want your final contig, the number of iterations (--end) can be changed. Usually, 4 iterations is sufficient for complete 18S or 28S. However, more or less can be used depending on the taxon and data. The default is set to 4 here.

Save the modified job below as "mitobim_loop.job" in the same directory that contains the trimmed reads files and submit the job on Hydra (qsub mitobim_loop.job).

When the job completes, there should be a separate folder labeled "mitobim_results" in the same directory as the job file. The final results for each sample will be in this directory. This final assembled (18S or 28S) contig will be in the subdirectory with the last assigned interation (iteration4 by default) and end with "_noIUPAC.fasta". 


```
# /bin/sh
# ----------------Parameters---------------------- #
#$ -S /bin/sh
#$ -pe mthread 3
#$ -q mThC.q
#$ -l mres=12G,h_data=4G,h_vmem=4G
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
mkdir -p mitobim_results
for GETSAMPLENAME in ./*_R1_PE_trimmed.fastq.gz
do
SAMPLENAME=$(basename "$GETSAMPLENAME" _R1_PE_trimmed.fastq.gz)
mkdir "${SAMPLENAME}_mitobim"
mv ./"${SAMPLENAME}_mitobim" ./mitobim_results
cd ./mitobim_results/"${SAMPLENAME}_mitobim" || exit
MITObim.pl \
-sample "${SAMPLENAME}" \
-ref name_of_project \
--readpool "full path to"/interleaved_sequences/${SAMPLENAME}_interleaved.fastq.gz \
--quick "full path to to fasta file with seed" \
--end 4 --pair --clean &> "log_${SAMPLENAME}"
cd ../..
done
#
echo = `date` job $JOB_NAME done

```

## Part 3 (Extra) Pull all final fasta contings and log files from directories and rename them with sample IDs
When the MITObim loop is complete (Part 2), the final contigs and log files of each sample will be in separate directories. This script will copy those files into a single directory and rename the final contig fasta files with SampleIDs.

1. Save the script below as "mitobim_rename.sh" and run the shell script (sh mitobim_rename.sh) in the "mitobim_results" directory created in Part 2. Explanations of the script steps are given in the text of the script.

2. Final renamed contigs files and log files will be copied to a directory called "mitobim_logs_and_final_contigs"

```
#!/bin/sh

# Run this script in the same directory that contains the final mitobim output files (*_mitobim)
# Creates a directory named mitobim_logs_and_final_contigs
mkdir -p mitobim_logs_and_final_contigs 

# Iterates over files/directories with names ending in _mitobim in the current directory
for mitobim_rename in *_mitobim; do
    # Prints the name of each file/directory being processed
    echo "Processing file: $mitobim_rename"

    # Copies files matching the pattern to the mitobim_logs_and_final_contigs directory
    cp ./${mitobim_rename}/iteration*/*_noIUPAC.fasta ./mitobim_logs_and_final_contigs
    cp ./${mitobim_rename}/log_* ./mitobim_logs_and_final_contigs
done

# Iterate over files in the directory
for mitobim_internalrename in ./mitobim_logs_and_final_contigs/*_noIUPAC.fasta; do
    # Extract filename without extension
    filename=$(basename "$mitobim_internalrename" .fasta)

    # Perform internal rename substitution and deletion using awk
    awk -v fname="$filename" -F '>' '{if (NF > 1) print $1 ">" fname; else print}' "$mitobim_internalrename" > tmpfile && mv tmpfile "$mitobim_internalrename"
done

echo "Final renamed contigs and logs should be in directory named 'mitobim_logs_and_final_contigs'"
echo "Check for any errors/warnings printed to screen"
echo "Done"

exit

```


# Assemble nuclear rDNA (18S and 28S) from genomic data with BBmap and MITObim using a loop
This workflow is in two parts. The first part will convert trimmed forward and reverse reads into single interleaved files uisng BBmap. The second part will use the interleaved data and a user provided seed to assemble nuclear ribosomal sequences using MITObim, rename the final contig files with sample IDs and save them to a new directory. 

## Part 1
## Convert trimmed reads into interleaved format using BBmap
Link to BBmap site - https://sourceforge.net/projects/bbmap/
### File preparation
The trimmed reads file names must end in "_R1_PE_trimmed.fastq.gz" and "_R2_PE_trimmed.fastq.gz" for the script to work. The script will utilize whatever text is in front of either "_R1(orR2)_PE_trimmed.fastq.gz" as the sample name. Alternatively, if your trimmed reads file names end with different text, the job file may also be modified accordingly. 

### Run BBmap on Hydra 
Before running the script, enter the full path to the trimmed sequences after "SAMPLEDIR_TRM=" in the job file below.
Verify that your trimmed reads file names end with the appropriate text, save the job file below as "bbmap_interleaved.job", and run the job below on Hydra (qsub bbmap_interleaved.job).

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
# Create directory where interleaved sequences will be placed
mkdir -p interleaved_sequences

# Create variable by copying full path to trimmed sequences
SAMPLEDIR_TRM="full path to trimmed sequences"

# Create variable by copying full path to the base directory.
# This is where the output directory will be located.
SAMPLEDIR_BASE="full path to base directory"

# Use loop to generate a sample names for each sample and run BBmap 
for GETSAMPLENAME in ${SAMPLEDIR_TRM}/*_R1_PE_trimmed.fastq.gz
do
SAMPLENAME=$(basename "$GETSAMPLENAME" _R1_PE_trimmed.fastq.gz)

reformat.sh \
in1="${SAMPLEDIR_TRM}/${SAMPLENAME}_R1_PE_trimmed.fastq.gz" \
in2="${SAMPLEDIR_TRM}/${SAMPLENAME}_R2_PE_trimmed.fastq.gz" \
out="${SAMPLEDIR_BASE}/interleaved_sequences/${SAMPLENAME}_interleaved.fastq.gz"
done

#
echo = `date` job $JOB_NAME done


```

## Part 2
## Use interleaved sequence data created in Part 1 to assemble nuclear rDNA using MITObim in a loop
link to MITObim - https://github.com/chrishah/MITObim
### File Preparation
Create a fasta file (.fasta) with the seed used for assembling the gene of interest. For nuclear ribosomal genes, partial or complete 18S or 28S sequences may be used. Depending on the taxon, using either 18S or 28S, MITObim may assemble the entire nuclear ribosomal operon, or may just assemble the single gene from the seed. 
 
### Run MITObim on Hydra
#### NOTE: The following commands in the job file must be modified before submitting to Hydra.

SAMPLEDIR_INT=

Paste full path to the directory that contains the interleaved sequences after the "=".

-ref name_of_project 

change "name_of_project" to the name of your project.

--quick 

After "--quick" paste the full path to the directory that contains the fasta file with the 18S or 28S seed.

--end 4 

#### Note: Depending on how large you want your final contig, the number of iterations (--end) can be changed. Usually, 4 iterations is sufficient for complete 18S or 28S. However, more or less can be used depending on the taxon and data. The default is set to 4 here.

At the end of the job file there is a script that will pull all final fasta contings (18S and/or 28S) and log files from the output directories, rename them with sample IDs, and copy them to a new directory named 'mitobim_final_renamed_contigs_and_logs'.

There will also be a separate directory labeled "mitobim_results" with raw final results for each sample. Here, the final assembled (18S or 28S) contig will be in the subdirectory with the last assigned interation (iteration4 by default) and end with "_noIUPAC.fasta". 

To run the MITObim job, save the modified job below as "mitobim_loop.job" and submit the job on Hydra (qsub mitobim_loop.job).


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

# Create variable by pasting full path to directory with interleaved sequences
SAMPLEDIR_INT="path to interleaved sequences"

# Use loop to generate sample names for each sample  
for GETSAMPLENAME in ${SAMPLEDIR_INT}/*_interleaved.fastq.gz
do
SAMPLENAME=$(basename "$GETSAMPLENAME" _interleaved.fastq.gz)

# Create sample-specific directories and move them to mitobim_results directory
mkdir "${SAMPLENAME}_mitobim"
mv ./"${SAMPLENAME}_mitobim" ./mitobim_results
cd ./mitobim_results/"${SAMPLENAME}_mitobim" || exit

# Run mitobim from sample-specific directories in a loop
MITObim.pl \
-sample "${SAMPLENAME}" \
-ref name_of_project \
--readpool "${SAMPLEDIR_INT}"/${SAMPLENAME}_interleaved.fastq.gz \
--quick "full path to to fasta file with seed" \
--end 4 --pair --clean &> "log_${SAMPLENAME}"
cd ../..
done

# Rename output files
# Creates a directory named mitobim_final_renamed_contigs_and_logs
mkdir -p mitobim_final_renamed_contigs_and_logs 

# Iterates over files/directories from mitobim outputs in the current directory
for mitobim_rename in ./mitobim_*/*_mitobim; do
    # Prints the name of each file/directory being processed
    echo "Processing file: $mitobim_rename"

    # Copies files matching the pattern to the mitobim_final_renamed_contigs_and_logs directory
    cp ./${mitobim_rename}/iteration*/*_noIUPAC.fasta ./mitobim_final_renamed_contigs_and_logs
    cp ./${mitobim_rename}/log_* ./mitobim_final_renamed_contigs_and_logs
done

# Iterate over files in the directory
for mitobim_internalrename in ./mitobim_final_renamed_contigs_and_logs/*_noIUPAC.fasta; do
    # Extract filename without extension
    filename=$(basename "$mitobim_internalrename" _noIUPAC.fasta)

    # Perform internal rename substitution and deletion using awk
    awk -v fname="$filename" -F '>' '{if (NF > 1) print $1 ">" fname; else print}' "$mitobim_internalrename" > tmpfile && mv tmpfile "$mitobim_internalrename"
done

echo "Final renamed contigs and logs should be in directory named 'mitobim_final_renamed_contigs_and_logs'"
echo "Check for any errors/warnings printed to screen"
echo "Done"

#
echo = `date` job $JOB_NAME done


```



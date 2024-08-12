# Assemble nuclear rDNA (18S and 28S) from genomic data using BBmap and MITObim using a loop
This workflow is in two parts. The first part will convert trimmed forward and reverse reads into a single interleaved file uisng BBmap. The second part will use the interleavd data and a user provided seed to assemble nuclear ribosomal sequences using MITObim.

## Part 1
## Convert trimmed reads into interleaved format using BBmap
Link to BBmap site - https://sourceforge.net/projects/bbmap/
### File preparation
1. Create a text file called "bbmap_samples.txt" with each sample ID in a column. Use unique sample IDs.

```
SAMPLE_ID_1
SAMPLE_ID_2
SAMPLE_ID_3
```

2. The trimmed reads file names for each sample should have the sample IDs from step 1 followed by "_R1_PE_trimmed.fastq.gz" and "_R2_PE_trimmed.fastq.gz" for the forward and reverse reads (R1 and R2).

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
## Use interleaved sequence data created in Part 1 to assemble nuclear rDNA using MITObim in a loop
link to MITObim - https://github.com/chrishah/MITObim
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

3. Run the job below in the same directory that contains the "mitobim_samples.txt" file (this path can be modified in the job file if desired). Since full paths are required for the "interleaved_sequences" directory and the seed fasta file, they do not have to be located in the same directory.

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
-ref name_of_project \
--readpool "full path to"/interleaved_sequences/${SAMPLE}_interleaved.fastq.gz \
--quick "full path to seed fasta file" \
--end 4 --pair --clean &> "log_${SAMPLE}"
cd ..
done
#
echo = `date` job $JOB_NAME done
```

## Part 3 (Extra) Pull all final fasta contings and log files from directories and rename them with sample IDs
When the MITObim loop is complete (Part 2), the final contigs and log files of each sample will be in separate directories. This script will copy those files into a single directory.
### For this script to work, lines 14 and 32 of the script below MUST be modified by the user
1. In line 14 the shell script below, modify the text "[DIGIT OF LAST ITERATION]" with the number of the last iteration run. This is the number entered for the flag "--end" in Part 2. For example, "iteration4".

2. In line 32 of the script below, modfy the awk command text "[FASTA_NAME_OF_SEED]" with the name of the seed used for the MITObim run. This is simply the name after the ">" of the fasta file used for the seed. If there are spaces in the name, just use the text up to the first space.

3. Save the script as "mitobim_rename.sh" and run the shell script (sh mitobim_rename.sh) in the same directory as the "[sample]_mitobim" directories from the output of MITObim from Part 2. Explanations of the script steps are given in the text of the script.

4. Final renamed contigs files and log files will be copied to a directory called "mitobim_logs_and_final_contigs"

```
#!/bin/sh

# Creates a directory named mitobim_final_contigs
mkdir mitobim_final_contigs_temp 

# Iterates over files/directories with names ending in _mitobim in the current directory
for mitobim_results in ./*_mitobim  

do

# Prints the name of each file/directory being processed
    echo "Processing file: $mitobim_results"  
# Copies files matching the pattern to the mitobim_final_contigs directory. Change "digit of last iteration" to appropriate value.
    cp ./${mitobim_results}/iteration[DIGIT_OF_LAST_ITERATION]/*_noIUPAC.fasta ./mitobim_final_contigs_temp  
    cp ./${mitobim_results}/log_* ./mitobim_final_contigs_temp
done

# Creates a directory named mitobim_final_contigs_internalrename
mkdir -p mitobim_logs_and_final_contigs

# Copy all files ending with _noIUPAC.fasta into the newly created directory
cp ./mitobim_final_contigs_temp/*_noIUPAC.fasta ./mitobim_logs_and_final_contigs
    
# Iterate over files in the directory
for mitobim_internalrename in ./mitobim_logs_and_final_contigs/*_noIUPAC.fasta 
do

    # Extract filename without extension
    filename=$(basename "$mitobim_internalrename" .fasta)

    # Perform substitution and deletion using awk - NOTE - Change "FASTA_NAME_OF_SEED" to the sample name used after the ">" in the fasta file for the seed in the MITObim run
    awk -v fname="$filename" '{gsub(/[FASTA_NAME_OF_SEED]/, fname); gsub(/_bb/, ""); print}' "$mitobim_internalrename" > tmpfile && mv tmpfile "$mitobim_internalrename"

done

mv ./mitobim_final_contigs_temp/log_* ./mitobim_logs_and_final_contigs

rm -r ./mitobim_final_contigs_temp

echo "Final renamed contigs and logs should be in directory named 'mitobim_logs_and_final_contigs'"
echo "Check for any errors/warnings printed to screen"
echo "Done"

exit

```


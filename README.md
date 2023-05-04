# Data preprocessing pipeline used for preparing the inputs of MERLIN-P-TFA in the GRAsp paper 
If you're a Roy Lab member, this GitHub repo is a mirror of the following folder:
```
/mnt/dv/wid/projects7/Roy-Aspergillus/merlin-preprocess/
```
The following instructions are compatible with Linux. We used CentOS Linux version 7. 

## Step 1: Download Raw Reads (.fastq.gz files)
For each dataset, identify its BioProject ID, e.g., for dataset GSE30579, the BioProject ID is PRJNA144647 as mentioned in Section "Relations" on its GEO page (https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE100101). Then go to SRA Explorer (https://sra-explorer.info/) and paste the BioProject ID. It will display all samples associated with the dataset. Select all the samples and add them to "collection".
**Warning:** Make sure the collection is empty before adding anything to collection. Sometimes SRA Explorer remembers your older sessions. To avoid that, refresh your browser before adding samples to the collection.

Once all the samples are added, a blue button with a shopping cart icon will show up at the top right corner. It will say something like "6 saved datasets" where 6 is the number of samples you have added to the collection (please don't get confused with the term "dataset"). Click on that button. It will open up a panel called "FastQ Downloads". The panel will have multiple collapsible sub-panels. Click on sub-panel "Bash script for downloading FastQ files". Copy the script and save it in a .sh file, say "fastq_downloader_SraExplorer.sh". Make it executable with "chmod +x fastq_downloader_SraExplorer.sh". Then run it with "bash fastq_downloader_SraExplorer.sh".

Once the raw reads are downloaded, separate the non-Aspergillus fumigatus samples from the Aspergillus ones. Only use the Aspergillus samples for further processing. Rename the .fastq.gz files if necessary. The SRA Explorer renames them unnecessarily. For single-end reads, simply rename them to their SRR ID, e.g., "SRR309223.fastq.gz". For paired-end reads, suffix them with 1 and 2, e.g., "SRR309220_1.fastq.gz" and "SRR309220_2.fastq.gz".

Download all sample accession IDs (e.g., SRR IDs) of Aspergillus fumigatus related to a dataset (say, GSE30579) and save inside a text file.
E.g., /mnt/dv/wid/projects7/Roy-Aspergillus/Data/RnaSeq/Ref_02/RoylabRsemProcessed/GSE30579/SRR_Acc_List_GSE30579.txt
One easy way to get the sample IDs is as follows:
Go to SRA Run Selector https://www.ncbi.nlm.nih.gov/Traces/study/?
* Enter the BioProject ID of the dataset (which is PRJNA144647 for dataset GSE30579)
* Under Section "Select", click on the "Accession List" button
* It will download a text file with all sample IDs
* remove non-Aspergillus fumigatus sample Ids
* rename the text file to "SRR_Acc_List_GSE30579.txt".

If there are single-end samples, create another text file named "SRR_Acc_List_GSE30579_SE.txt" containing only the single-end sample IDs.
If there are paired-end samples, create another text file named "SRR_Acc_List_GSE30579_PE.txt" containing only the paired-end sample IDs.

For example, GSE30579 has total 4 samples: 2 single-end and 2 paired-end. Therefore, there are three text files.
SRR_Acc_List_GSE30579.txt: contains all 4 sample IDs.
SRR_Acc_List_GSE30579_SE.txt: contains 2 sample IDs that are single-ended.
SRR_Acc_List_GSE30579_PE.txt: contains 2 sample IDs that are paired-ended.
These text files will be indispensable while running scripts in the future steps.

---
## Step 2: Clip Adapter Sequences with Trimmomatic
Apply Trimmomatic on the raw reads to trim adapter sequences.

For single-end reads, use the following command:
```
## Trimmomatic reads the raw reads from ${RAW_READ_DIR} and outputs to ${TRIMMED_READ_DIR}).
## TRIMMOMATIC_DIR="programs/Trimmomatic-0.39/Trimmomatic-0.39"
java -jar ${TRIMMOMATIC_DIR}/trimmomatic-0.39.jar SE -trimlog ${TRIMMED_READ_DIR}/${SRR_ID}_trimlog.txt ${RAW_READ_DIR}/${SRR_ID}.fastq.gz ${TRIMMED_READ_DIR}/${SRR_ID}.fq.gz ILLUMINACLIP:${TRIMMOMATIC_DIR}/adapters/.fa:2:30:10:2:keepBothReads LEADING:5 TRAILING:5 MINLEN:36
```

For paired-end reads, use the following command:
```
java -jar ${TRIMMOMATIC_DIR}/trimmomatic-0.39.jar PE -trimlog ${TRIMMED_READ_DIR}/${SRR_ID}_trimlog.txt ${RAW_READ_DIR}/${SRR_ID}_1.fastq.gz ${RAW_READ_DIR}/${SRR_ID}_2.fastq.gz ${TRIMMED_READ_DIR}/${SRR_ID}_forward_paired.fq.gz ${TRIMMED_READ_DIR}/${SRR_ID}_forward_unpaired.fq.gz ${TRIMMED_READ_DIR}/${SRR_ID}_reverse_paired.fq.gz ${TRIMMED_READ_DIR}/${SRR_ID}_reverse_unpaired.fq.gz ILLUMINACLIP:${TRIMMOMATIC_DIR}/adapters/.fa:2:30:10:2:keepBothReads LEADING:5 TRAILING:5 MINLEN:36
```

Selecting the appropriate adapter file is critical. Please read Section "The Adapter Fasta" in the Trimmomatic manual to get a general guideline (programs/Trimmomatic-0.39/TrimmomaticManual_V0.32.pdf). Subsequently, read Section "Materials and Methods" of the reference paper for the concerned dataset to check whether there are any deviations from the general guideline. Please also read the discussion at https://www.biostars.org/p/323087/ to understand how other researcher select adapter files.

**Q. Which adapter FASTA file to use with the in-house dataset GSE231238 (a.k.a. Ref_39)?**  
**A.** The RNA sequencing for this dataset had been performed by a company named Novogene.
However, when we asked Novogene which adapter version had been used, they remarked that they do not possess that record anymore. Fortunately,
we were able to find out the exact adapter sequences Novogene used from the Novogene delivery documents. By comparing these sequences with
that of different adapter files, "TruSeq2-PE.fa" is found to be the best match. It has a perfect match for the 5' adapter and an exact reverse
complement for the 3' adapter. The reverse complement part remains the sole source of confusion. However, we decided to go ahead with
"TruSeq2-PE.fa" for these data sets.
The Novogene adapter sequences are shown in the figure below.
![Figure not found!](figures/210816_153257_novogene_adapter_seqences.png)

Different versions of adapter files are retrieved from https://github.com/usadellab/Trimmomatic/tree/main/adapters.

---
## Step 3: Generate Quality Control (QC) Reports with FastQC-MultiQC
For each dataset, generate FastQC reports using the following command:
```
## Use 8 parallel threads
programs/FastQC/fastqc -t 8 <input_dir>/*.fastq.gz -o <output_dir>/
```

Combine all the FastQC reports corresponding to the given dataset into a single MultiQC report, we need the "multiqc" command.
To install the multiqc command, please create a conda environment -> install multiqc -> activate the environment.
```
## If you're a Roy Lab member, first, load the anaconda module:
# module load anaconda3-2020.02

conda create --name multiqc python=3.7.6
conda install -c bioconda -c conda-forge multiqc
conda activate multiqc
```

Once the multiqc command is installed, run it as follows:
```
multiqc -o <output_dir>/ <input_dir>/
```

For each dataset, generate one MultiQC report before trimming (i.e. with raw reads/.fastq.gz files) and one MultiQC report after trimming (i.e. with trimmed reads/.fq.gz files). Analyze whether the trimming was effective by comparing the two reports. Please see https://elog.discovery.wisc.edu/Software/165 to learn how to interpret a MultiQC report.

---

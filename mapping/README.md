Pipeline for mappability filtering

###############
Step 1: Map the fastq files using your favorite mapper/options and filter for quality using a cutoff of your choice

Example:
tophat --no-coverage-search -o ${LANE_NAME}_out Sequences/hg18_norand ${LANE_NAME}.fastq.gz
samtools view -b -q 10 ${LANE_NAME}_out/accepted_hits.bam > ${LANE_NAME}_out/accepted_hits.quality.bam

################
Step 2: Use find_intersecting_snps.py to identify reads that may have mapping biases
Usage: 
find_intersecting_snps.py [-p] <input.bam> <SNP_file_directory>

-p option indicates that reads are paired-end (default is single)

-m # option changes the maximum window to search for SNPs.  The default is 100,000 base pairs.  Reads or read pairs that span more than this distance (usually do to splice junctions) will be thrown out.  Increasing this window allows for longer junctions, but may increase run time and memory requirements.

<input.bam> is the bamfile from the initial mapping process

<SNP_file_directory> is the directory containing the SNPs segregating within the sample in question (which need to be checked for mappability issues).  This directory should contain sorted files of SNPs separated by chromosome and named:
chr<#>.snps.txt.gz

These files should contain 3 columns:
position RefAllele AltAllele

Output:
input.sort.bam-
	Sorted bamfile of the original input
input.keep.bam-
	bamfile with reads that did not intersect SNPs and therefore can be kept without remapping
input.to.remap.bam-
	bamfile with original reads that need to be remapped
input.to.remap.num.gz-
	matched lines with the input.to.remap.bam that indicate the number of variants of the original read that must be remapped
input.remap.fq.gz-
	fastq file containing the new variants to remap
	will be .fq1.gz and .fq2.gz if the paired end option is used

Example:
python find_intersecting_snps.py ${LANE_NAME}_out/accepted_hits.quality.bam SNP_files/

Step 3
Map the input.remap.fq.gz using the same mapping choices as before

Example:
tophat --no-coverage-search -o ${LANE_NAME}_out_remap hg18_norand ${LANE_NAME}_out/accepted_hits.quality.remap.fq.gz
samtools view -b -q 10 ${LANE_NAME}_out_remap/accepted_hits.bam > ${LANE_NAME}_out_remap/accepted_hits.quality.bam

Step 4
Use filter_remapped_reads.py to retrieve reads that remapped correctly

Usage:
filter_remapped_reads.py [-p] <to.remap.bam> <remapped_reads.bam> <output.bam> <to.remap.num.gz>

-p option indicates that the reads are paired-end

<to.remap.bam> is the file output from find_intersecting_snps.py which contains the original aligned reads that were remapped
<remapped_reads.bam> is the output from the second mapping step (Step 3)
<output.bam> is the file where reads that are kept after remapping are stored
<to.remap.num.gz> is the file from find_intersecting_snps.py which contains the number of remapped sequences

Example:
filter_remapped_reads.py ${LANE_NAME}_out/accepted_hits.quality.to.remap.bam ${LANE_NAME}_out_remap/accepted_hits.quality.bam ${LANE_NAME}.remap.keep.bam ${LANE_NAME}_out/accepted_hits.quality.to.remap.num.gz

At the end of the pipeline, ${LANE_NAME}.keep.bam and ${LANE_NAME}.remap.keep.bam can be merged for a complete set of mappability filtered aligned reads
	
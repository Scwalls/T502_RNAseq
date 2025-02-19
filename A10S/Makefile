# Makefile_GoSTRIPES
#
# Version: October 15, 2019.
#

# Note that it is assumed that all required executables are in your $PATH.
# If you are using Singularity to run the GoSTRIPES workflow, then this is
# taken care of. Otherwise, refer to the Singularity recipe on how to set up
# a proper GoSTRIPES environment.

# Typical customization should only involve filling in the variables in the
# next section.

################################################################################
####################### ! Variable Settings ! ##################################
################################################################################

# This template Makefile is ready to run the demo described on the GoSTRIPES
# repository site. To run your own data, you should edit the RAWREAD* variables
# to point to data source and optionally adjust other parameters to fit your
# system and preferences.

BASEDIR  = ${CURDIR}/..
RAWREAD1 = A10S_R1
RAWREAD2 = A10S_R2
RAWREAD1FQ = ${BASEDIR}/downloads/${RAWREAD1}.fq
RAWREAD2FQ = ${BASEDIR}/downloads/${RAWREAD2}.fq

UMI     = NNNNNNNN
LINKER  = TATAGGG
PATTERN = ${UMI}${LINKER}
SAMPLE  = A10S

# Genome file locations:

GENOME_DIR  = ${BASEDIR}/DpGENOME
GENOME_FILE = ${GENOME_DIR}/PA42.4.1.fasta
GENOME_GTF  = ${GENOME_DIR}/PA42.4.0.gff
RNAfile     = ${GENOME_DIR}/Dpul-rDNA.fa


# Options for programs called in the GoSTRIPES workflow:
#
FASTQC_OPTIONS        = --threads 2  --extract

TRIMMOMATIC_THREADS   = 6
TrueSeq2PE            = /opt/Trimmomatic-0.39/adapters/TruSeq2-PE.fa

TAGDUST_THREADS       = 6
TAGDUST_OPTIONS       = -dust 97 -t ${TAGDUST_THREADS} -fe 3

MIN_READ_LENGTH       = 25
CUTADAPT_THREADS      = 6

STAR_INDEX_OPTIONS    = --genomeSAindexNbases 10 --sjdbGTFfile ${GENOME_GTF}
STAR_THREADS          = 6

SAMTOOLS_THREADS      = 6
SAMTOOLS_SORT_OPTIONS = -m 7G -@ ${SAMTOOLS_THREADS}

scrubSAMfile_OPTIONS  = -s 3 -i 50 -I 1000 -t 1500

################################################################################
### ! Typically there would be no need for further editing below this line ! ###
################################################################################

all:	setup fastqc1 fastqc2 \
	${SAMPLE}_trimmed_R1.fq ${SAMPLE}_trimmed_R2.fq \
	${SAMPLE}_unpaired_R1.fq ${SAMPLE}_unpaired_R2.fq \
	${SAMPLE}_tagdusted_READ1.fq ${SAMPLE}_tagdusted_READ2.fq tmp_${SAMPLE}.fastq \
	${SAMPLE}_tagdusted_un_READ1.fq ${SAMPLE}_tagdusted_un_READ2.fq \
	${SAMPLE}_umi_R1.fq ${SAMPLE}_umi_R2.fq \
	${SAMPLE}_select_R1.fq ${SAMPLE}_select_R2.fq \
	${SAMPLE}_noa_R1.fq ${SAMPLE}_noa_R2.fq \
	${SAMPLE}_nob_R1.fq ${SAMPLE}_nob_R2.fq \
	trm_${SAMPLE}.fastq unt_${SAMPLE}.fastq \
	${SAMPLE}_nod.fastq \
	${SAMPLE}_clean.fastq \
	${SAMPLE}_proper.bam \
	${SAMPLE}_clean_R1.fq ${SAMPLE}_clean_R2.fq ${SAMPLE}.stats \
	alignments/${SAMPLE}.sam \
	alignments/Scrubbed-${SAMPLE}.sam \
	${SAMPLE}.bam

.PHONY:	setup


setup:
ifeq ("$(wildcard ./FastQC)","")
	mkdir FastQC
endif


### Run FastQC to check the quality of the raw reads.
#
fastqc1:	setup FastQC/${SAMPLE}_raw_R1_fastqc/fastqc_data.txt FastQC/${SAMPLE}_raw_R2_fastqc/fastqc_data.txt

FastQC/${SAMPLE}_raw_R1_fastqc/fastqc_data.txt:	${SAMPLE}_raw_R1.fq
	fastqc  ${FASTQC_OPTIONS}  --outdir=FastQC  ${SAMPLE}_raw_R1.fq

FastQC/${SAMPLE}_raw_R2_fastqc/fastqc_data.txt:	${SAMPLE}_raw_R2.fq
	fastqc  ${FASTQC_OPTIONS}  --outdir=FastQC  ${SAMPLE}_raw_R2.fq

${SAMPLE}_raw_R1.fq:	${RAWREAD1FQ}
ifeq ("$(wildcard ${SAMPLE}_raw_R1.fq)","")
	if [ $(shell file ${RAWREAD1FQ} | egrep "gzip" | wc -l) -gt 0 ]; then \
		gunzip -c ${RAWREAD1FQ} > ${SAMPLE}_raw_R1.fq; \
	else \
		ln -rs ${RAWREAD1FQ} ${SAMPLE}_raw_R1.fq; \
	fi
endif

${SAMPLE}_raw_R2.fq:	${RAWREAD2FQ}
ifeq ("$(wildcard ${SAMPLE}_raw_R2.fq)","")
	if [ $(shell file ${RAWREAD2FQ} | egrep "gzip" | wc -l) -gt 0 ]; then \
		gunzip -c ${RAWREAD2FQ} > ${SAMPLE}_raw_R2.fq; \
	else \
		ln -rs ${RAWREAD2FQ} ${SAMPLE}_raw_R2.fq; \
	fi
endif


### Trimming the reads and removing reads that match rRNA targets.
#
%_trimmed_R1.fq %_trimmed_R2.fq %_unpaired_R1.fq %_unpaired_R2.fq:	%_raw_R1.fq %_raw_R2.fq
	java -jar /opt/Trimmomatic-0.39/trimmomatic-0.39.jar PE \
	-threads ${TRIMMOMATIC_THREADS} \
	$*_raw_R1.fq $*_raw_R2.fq \
	$*_trimmed_R1.fq $*_unpaired_R1.fq $*_trimmed_R2.fq $*_unpaired_R2.fq \
	ILLUMINACLIP:${TrueSeq2PE}:2:30:10 TRAILING:20 MINLEN:25

%_tagdusted_READ1.fq %_tagdusted_READ2.fq %_tagdusted_un_READ1.fq %_tagdusted_un_READ2.fq:	%_trimmed_R1.fq %_trimmed_R2.fq
	tagdust -ref ${RNAfile} ${TAGDUST_OPTIONS} -1 R:N $*_trimmed_R1.fq $*_trimmed_R2.fq  -o $*_tagdusted


### Getting reads matching the specified PATTERN and removing the tag sequences.
#
%_select_R1.fq %_select_R2.fq tmp_%.fastq:	%_tagdusted_READ1.fq %_tagdusted_READ2.fq
	fastq-interleave $*_tagdusted_READ1.fq $*_tagdusted_READ2.fq > tmp_${SAMPLE}.fastq
	selectReadsByPattern.pl -m ${PATTERN} -p -o ${SAMPLE}_select -r tmp_${SAMPLE}.fastq > select_$*_tagdusted_logfile.txt

%_umi_R1.fq %_umi_R2.fq:	%_select_R1.fq %_select_R2.fq
	umi_tools extract -I $*_select_R1.fq -p ${UMI} --read2-in=$*_select_R2.fq -S $*_umi_R1.fq --read2-out=$*_umi_R2.fq

%_noa_R1.fq %_noa_R2.fq:	%_umi_R1.fq %_umi_R2.fq
	cutadapt -j ${CUTADAPT_THREADS} -m ${MIN_READ_LENGTH} -g ^${LINKER} -e 0 \
		-o $*_noa_R1.fq -p $*_noa_R2.fq  $*_umi_R1.fq $*_umi_R2.fq > cutadapter_noa_logfile.txt


### More scrubbing:
#
%_nob_R1.fq %_nob_R2.fq:	%_noa_R1.fq %_noa_R2.fq
	cutadapt -j ${CUTADAPT_THREADS} --front=CCTACACGACGCTCTTCCGATCT --overlap=10 --error-rate=0.09 \
		-o $*_nob_R1.fq -p $*_nob_R2.fq $*_noa_R1.fq $*_noa_R2.fq > cutadapter_nob_logfile.txt

trm_%.fastq unt_%.fastq:	%_noa_R1.fq %_nob_R1.fq %_noa_R2.fq
	selectAdapterTrimmedReads.pl -f $*_noa_R1.fq -t $*_nob_R1.fq -r $*_noa_R2.fq -o $* > adapter_trimming_nob_logfile.txt

%_nod.fastq:	trm_%.fastq
	if [ -s trm_$*.fastq ]; then \
	  selectReadsByPattern.pl -m ${PATTERN} -p -i -o $*_noc.fastq -r trm_$*.fastq > select_trm_$*_logfile.txt; \
	  fastq-deinterleave $*_noc; \
	  umi_tools extract -I $*_noc_R1.fq -p ${UMI} --read2-in=$*_noc_R2.fq -S $*_nocumi_R1.fq --read2-out=$*_nocumi_R2.fq; \
	  cutadapt -j ${CUTADAPT_THREADS} -m ${MIN_READ_LENGTH} -g ^${LINKER} -e 0 \
		-o $*_nod_R1.fq -p $*_nod_R2.fq  $*_nocumi_R1.fq $*_nocumi_R2.fq > cutadapter_nod_logfile.txt; \
	  fastq-interleave $*_nod_R1.fq $*_nod_R2.fq > $*_nod.fastq; \
	else \
 	  touch $*_nod.fastq; \
	fi

%_clean.fastq %_clean_R1.fq %_clean_R2.fq:	%_nod.fastq unt_%.fastq
	cat $*_nod.fastq unt_$*.fastq > $*_clean.fastq
	fastq-deinterleave $*_clean


### FastQC again to check the quality of the trimmed and tagdusted reads.
#
fastqc2:	FastQC/${SAMPLE}_clean_R1_fastqc/fastqc_data.txt FastQC/${SAMPLE}_clean_R2_fastqc/fastqc_data.txt

FastQC/${SAMPLE}_clean_R1_fastqc/fastqc_data.txt:	${SAMPLE}_clean_R1.fq
	fastqc  ${FASTQC_OPTIONS}  --outdir=FastQC  ${SAMPLE}_clean_R1.fq

FastQC/${SAMPLE}_clean_R2_fastqc/fastqc_data.txt:	${SAMPLE}_clean_R2.fq
	fastqc  ${FASTQC_OPTIONS}  --outdir=FastQC  ${SAMPLE}_clean_R2.fq


### Generate and print basic sample statistics.
#
${SAMPLE}.stats:	${SAMPLE}_raw_R1.fq ${SAMPLE}_raw_R2.fq \
			FastQC/${SAMPLE}_raw_R1_fastqc/fastqc_data.txt FastQC/${SAMPLE}_raw_R2_fastqc/fastqc_data.txt \
 			${SAMPLE}_clean_R1.fq ${SAMPLE}_clean_R2.fq \
 			FastQC/${SAMPLE}_clean_R1_fastqc/fastqc_data.txt FastQC/${SAMPLE}_clean_R2_fastqc/fastqc_data.txt
	sstats-pe.sh ${SAMPLE}


### STAR alignment.
#

${GENOME_DIR}/STAR/SAindex:	${GENOME_FILE}
ifeq ("$(wildcard ${GENOME_DIR}/STAR)","")
	mkdir ${GENOME_DIR}/STAR
endif
	cd ${GENOME_DIR}/STAR && \
	STAR --runMode genomeGenerate --runThreadN ${STAR_THREADS} ${STAR_INDEX_OPTIONS} --genomeDir ${GENOME_DIR}/STAR --genomeFastaFiles ${GENOME_FILE}

STARdir/${SAMPLE}_STAR_Aligned.sortedByCoord.out.bam:	${GENOME_DIR}/STAR/SAindex  ${SAMPLE}_clean_R1.fq ${SAMPLE}_clean_R2.fq
ifeq ("$(wildcard ./STARdir)","")
	mkdir STARdir
endif
	cd STARdir && \
	STAR --runMode alignReads --limitBAMsortRAM 19114876484 --runThreadN ${STAR_THREADS} \
		--outSAMtype BAM SortedByCoordinate --outSAMorder Paired  --outFileNamePrefix ${SAMPLE}_STAR_ \
		--genomeDir ${GENOME_DIR}/STAR  --readFilesIn ../${SAMPLE}_clean_R1.fq ../${SAMPLE}_clean_R2.fq

${SAMPLE}_proper.bam:	STARdir/${SAMPLE}_STAR_Aligned.sortedByCoord.out.bam
	samtools view -b -f 2 -F 256 -@ ${SAMTOOLS_THREADS} STARdir/${SAMPLE}_STAR_Aligned.sortedByCoord.out.bam > ${SAMPLE}_proper.bam

alignments/${SAMPLE}.bam:	${SAMPLE}_proper.bam
ifeq ("$(wildcard ./alignments)","")
	mkdir alignments
endif
	samtools sort ${SAMTOOLS_SORT_OPTIONS} -n ${SAMPLE}_proper.bam | \
	samtools fixmate -rm - - | samtools sort - | samtools markdup -r - alignments/${SAMPLE}.bam

alignments/${SAMPLE}.sam:	alignments/${SAMPLE}.bam
	samtools sort ${SAMTOOLS_SORT_OPTIONS} -n -O SAM alignments/${SAMPLE}.bam > alignments/${SAMPLE}.sam

alignments/Scrubbed-${SAMPLE}.sam:	alignments/${SAMPLE}.sam
	cd alignments && scrubSAMfile.pl ${scrubSAMfile_OPTIONS} -o ${SAMPLE}.sam -r ${SAMPLE}.sam \
		> scrubbing_${SAMPLE}_logfile.txt

${SAMPLE}.bam:	alignments/Scrubbed-${SAMPLE}.sam
	samtools view -H alignments/${SAMPLE}.sam > header
	cat header alignments/Scrubbed-${SAMPLE}.sam | samtools sort | samtools view -bh > ${SAMPLE}.bam
	samtools index ${SAMPLE}.bam
	echo "" && \
	echo "All done." && echo "Check the output in 'alignments' and optionally clean up with" && \
	echo "	make cleanup" && \
	echo "	make finishup"



### Clean up the output directory.
#
cleanup:
	-\mkdir _SCRATCH
	-\mv ${SAMPLE}_trimmed_R1.fq ${SAMPLE}_trimmed_R2.fq _SCRATCH/
	-\mv ${SAMPLE}_unpaired_R1.fq ${SAMPLE}_unpaired_R2.fq _SCRATCH/
	-\mv ${SAMPLE}_tagdusted_READ1.fq ${SAMPLE}_tagdusted_READ2.fq tmp_${SAMPLE}.fastq _SCRATCH/
	-\mv ${SAMPLE}_tagdusted_un_READ1.fq ${SAMPLE}_tagdusted_un_READ2.fq _SCRATCH/
	-\mv ${SAMPLE}_umi_R1.fq ${SAMPLE}_umi_R2.fq _SCRATCH/
	-\mv ${SAMPLE}_select_R1.fq ${SAMPLE}_select_R2.fq _SCRATCH/
	-\mv ${SAMPLE}_noa_R1.fq ${SAMPLE}_noa_R2.fq _SCRATCH/
	-\mv ${SAMPLE}_nob_R1.fq ${SAMPLE}_nob_R2.fq _SCRATCH/
	-\mv trm_${SAMPLE}.fastq unt_${SAMPLE}.fastq _SCRATCH/
	-\mv ${SAMPLE}_noc.fastq ${SAMPLE}_noc_R1.fq ${SAMPLE}_noc_R2.fq _SCRATCH/
	-\mv ${SAMPLE}_nocumi_R1.fq ${SAMPLE}_nocumi_R2.fq _SCRATCH/
	-\mv ${SAMPLE}_nod.fastq ${SAMPLE}_nod_R1.fq ${SAMPLE}_nod_R2.fq _SCRATCH/
	-\mv ${SAMPLE}_clean.fastq _SCRATCH/
	-\mv header _SCRATCH/
	-\mkdir logfiles
	-\mv STARdir/${SAMPLE}_STAR_Log.final.out logfiles/
	-\mkdir STORE-${SAMPLE}
	-\mv FastQC STARdir STORE-${SAMPLE}/
	-\mv ${SAMPLE}_clean_R1.fq ${SAMPLE}_clean_R2.fq STORE-${SAMPLE}/
	-\mv ${SAMPLE}_proper.bam STORE-${SAMPLE}/
	-\mv ${SAMPLE}_tagdusted_logfile.txt logfiles/
	-\mv select*logfile.txt logfiles/
	-\mv adapter_trimming*logfile.txt cutadapter*logfile.txt logfiles/
	-\mv ${SAMPLE}.stats logfiles/
	-\mv alignments/scrubbing_${SAMPLE}_logfile.txt logfiles/

### Finish up after cleanup.
#
finishup:
	-\rm -rf _SCRATCH
	zip -r STORE-${SAMPLE} STORE-${SAMPLE}
	-\rm -rf STORE-${SAMPLE}

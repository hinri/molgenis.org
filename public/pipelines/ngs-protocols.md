# Preprocessing (aligning --> bam )
### Step 1: Spike in the PhiX reads

To see whether the pipeline ran correctly. The reads will be inserted in each sample. Later on (step 23) of the pipeline there will be a concordance check to see if the SNPs that are put in, will be found.

Scriptname: SpikePhiX

Input: raw sequence file in the form of a gzipped fastq file

Output: FastQ files (${filePrefix}_${lane}_${barcode}.fq.gz)*

### Step 2: Check the Illumina encoding

In this step the encoding of the FastQ files will be checked. Older (2012 and older) sequence data contains the old Phred+64 encoding (this is called Illumina 1.5 encoding), new sequence data is encoded in Illumina 1.8 or 1.9 (Phred+33). If the data is 1.5, it will be converted to 1.9 encoding

Scriptname: CheckIlluminaEncoding
Input:. FastQ files (${filePrefix}_${lane}_${barcode}.fq.gz)*

### Step 3: Calculate QC metrics on raw data

In this step, Fastqc, quality control (QC) metrics are calculated for the raw sequencing data. This is done using the tool FastQC. This tool will run a series of tests on the input file. The output is a text file containing the output data which is used to create a summary in the form of several HTML pages with graphs for each test. Both the text file and the HTML document provide a flag for each test: pass, warning or fail. This flag is based on criteria set by the makers of this tool. Warnings or even failures do not necessarily mean that there is a problem with the data, only that it is unusual compared to the used criteria. It is possible that the biological nature of the sample means that this particular bias is to be expected.

Toolname: FastQC

Scriptname: Fastqc

Input: FastQ files (${filePrefix}_${lane}_${barcode}.fq.gz)

Output: ${filePrefix}.fastqc.zip archive containing amongst others the HTML document and the text file

```
fastqc \
	fastq1.gz \
	fastq2.gz \
	-o outputDirectory
```
### Step 4: Read alignment against reference sequence

In this step, the Burrows-Wheeler Aligner (BWA) is used to align the (mostly paired end) sequencing data to the reference genome. The method that is used is BWA mem. The output is a SAM file.

Scriptname: BwaAlign

Input: raw sequence file in the form of a gzipped fastq file (${filePrefix}.fq.gz)
Output: SAM formatted file (${filePrefix}.sam)

Toolname: BWA

```
bwa mem \
    -M \
    -R $READGROUPLINE \
    -t 8 \
	-human_g1k_v37.fasta \
	-fastq1.gz \
	-fastq2.gz \
	> output.sam
	
```
### Step 5: Convert SAM to BAM

Using the Picard SamFormatConverter the SAM file generated by the previous step is converted to a compressed binary format (BAM).

Toolname: Picard SamFormatConverter

Scriptname: SamToBam

Input: SAM file generated in step 4 (${filePrefix}.sam)

Output: compressed binary (BAM) format (${filePrefix}.bam)

```
java -jar -Xmx3g picard.jar SamFormatConverter \
	INPUT= input.sam \
	OUTPUT= output.bam \
	VALIDATION_STRINGENCY=LENIENT \
	MAX_RECORDS_IN_RAM=2000000 
```
### Step 6: Sort BAM and build index

The reads in the BAM file are sorted (coordinate based) using Sambamba sort . An index file is generated automatically, this allows for efficient random access to the BAM file and is used by many tools to speed up reading from a BAM file.

Toolname: Sambamba sort

Scriptname: SortBam

Input: BAM file from step 5

Output:
- Sorted BAM file (${sample}.sorted.bam)
- Index file (${sample}.sorted.bai)

```
sambamba sort \
	-t 10 \
	-m 4GB \
	-o output.bam \
	input.bam
```
### Step 7: Merge BAMs and build index

To improve the coverage of sequence alignments, a sample can be sequenced on multiple lanes and/or flowcells. If this is the case for the sample(s) being analyzed, this step merges all BAM files of one sample and indexes this new file. If there is just one BAM file for a sample, nothing happens.

Toolname: Sambamba merge

Scriptname: SambambaMerge

Input: BAM files from step 6 (${sample}.sorted.bam)

Output: merged BAM file (${sample}.merged.bam)

```
sambamba merge \
	output.bam \
	${ARRAYOFINPUTBAMS[@]}
```
### Step 8: Mark duplicates + creating dedup metrics

In this step, the BAM file is examined to locate duplicate reads, using Sambamba markdup. A mapped read is considered to be duplicate if the start and end base of two or more reads are located at the same chromosomal position in comparison to the reference genome. For paired-end data the start and end locations for both ends need to be the same to be called duplicate. One read pair of these duplicates is kept, the remaining ones are flagged as being duplicate.

Toolname: Sambamba markdup & Sambamba flagstat

Scriptname: MarkDuplicates

Input: Merged BAM file from generated in step 7 (${sample}.merged.bam)

Output:
- BAM file with duplicates flagged (${sample}.dedup.bam)
- BAM index file (${sample}.dedup.bam.bai)
- Dedup metrics file (${sample}.merged.dedup.metrics)

Mark duplicates:
```
sambamba markdup \
	--nthreads=4 \
	--overflow-list-size 1000000 \
	--hash-table-size 1000000 \
	-p \
	input.bam \
	output.bam
```

dedup metrics:
```
sambamba flagstat \
	--nthreads=4 \
	input.bam \
	> output.flagstat
```
# Indel calling with Delly
### Step 09a: Calling big deletions with Delly

In this step, the progam Delly calls deletions from the merged BAM file. The deletions are written to a VCF file, along with information such as difference in length between REF and ALT alleles, type of structural variant end information about allele depth.

Toolname: Delly

Scriptname: Delly

Input: merged BAM file from step 7 (${sample}.merged.bam)
Output: indels in VCF (${sample}.delly.vcf)

```
delly \
	-n \
	-t DEL \
	-x human.hg19.excl.tsv \
	-o outputDelly.vcf \
	-g human_g1k_v37.fa \
	input.bam
```
### Step 09b. Delly annotator

There will be some annotation to the produced vcf by Delly. This step will add hpo terms and snpEff annotation

Toolname: CmdAnnotator

Scriptname: DellyAnnotator

Input: indels in VCF (${sample}.delly.vcf)

Output:
- ${sample}.delly.snpeff.vcf
- ${sample}.snpeff.hpo.vcf

snpEff:
```
java -Xmx10g -jar CmdLineAnnotator-1.9.0.jar \
	snpEff \
	snpEff.jar \
	outputDelly.vcf \
	snpEffAnnotatedoutputDelly.vcf
```

HPO:
```
java -Xmx10g -jar CmdLineAnnotator-1.9.0.jar \
	hpo \
	ALL_SOURCES_TYPICAL_FEATURES_diseases_to_genes_to_phenotypes.txt \
	snpEffoutputDelly.vcf \
	snpEff_and_HPO_AnnotatedoutputDelly.vcf
```
# Determine gender

### Step 10a: GenderCalculate

Due to the fact a male has only one X chromosome it is important to know if the sample is male or female. Calculating the coverage on the non pseudo autosomal region and compare this to the average coverage on the complete genome predicts male or female well.

Toolname: Picard CalculateHSMetrics
Scriptname: GenderCalculate

Input: dedup BAM file (${sample}.dedup.bam)

Output: ${dedupBam}.nonAutosomalRegionChrX_hs_metrics

```
java -jar -Xmx4g picard.jar CalculateHsMetrics \
	INPUT=input.bam \
	TARGET_INTERVALS=input.nonAutosomalChrX.interval_list \
	BAIT_INTERVALS=input.nonAutosomalChrX.interval_list \
	OUTPUT=output.nonAutosomalRegionChrX_hs_metrics
```
# Side steps (Cram conversion and concordance check)
### Step 10b: CramConversion

Producing more compressed bam files, decreasing size with 40%

Scriptname: CramConversion
Toolname: Scramble

Input: dedup BAM file (${sample}.dedup.bam)

Output: dedup CRAM file (${sample}.dedup.bam.cram)

```
scramble \
	-I bam \
	-O cram \
	-r human_g1k_v37.fa \
	-m \
	-t 8 \
	input.bam \
	output.cram
```

### Step 11: Make md5’s for the realigned bams

Small step to create md5sums for the realigned bams created in step 9

Input: realigned BAM file (.realigned.bam)

Output: md5sums (.realigned.bam.md5)

### Step 12: SequonomConcordance Check

Scriptname: SequenomConcordanceCheck

As a last measure of quality of the SNPs reported in the previous steps, the concordance between the SNPs and the SNPs called using a Sequenom of the same sample is checked. If at least … SNPs overlap and the concordance is around 97% or higher, the SNPs reported in the previous steps are accepted as being of reliable quality and exclude a potential sample swap.

The concordance check is done in several steps (provided Sequenom report is present):

● sed, awk, uniq, sort, grep to convert the Sequenom report of an analyzed sample into map, lgen and fam files (plink long-format fileset)

● plink-1.07 recode which is used for creating a *.bed file

● plink 1.08 unpublished development version recode which is used to create a genotype in *.vcf format

● command line Perl to change the header

● sed and awk to convert from vcf to bed

● fastaFromBed from BEDTools-Version-2.11.2 to create a uscs style tab elimited fasta file from the bed file, using reference sequence build 37

● align-vcf-to-ref.pl from inhouse_scripts

● head, sed, cat to change header

● iChip_pos_to_interval_list.pl from inhouse_scripts is used to create an interval list of Sequenom SNPs which is used to call inhouse SNPs

● GATK-1.2-1-g33967a4 UnifiedGenotyper which is used for calling SNPs on all positions known to be on array and VCF and calculating the concordance between array SNPs and inhouse pipeline SNPs

● change_vcf_filter.pl from inhouse scripts is used to change the FILTER column from GATK "called SNPs". All SNPs having Q20 & DP10 change to "PASS", all other SNPs are "filtered" (not used in concordance check)

● GATK-1.2-1-g33967a4 VariantEval to calculate condordance between Sequenom SNPs and GATK "called SNPs"

● echo to prepare the header of the output concordance file

● R script extract_info_GATK_variantEval_V3.R (using library from GATK-1.3-24-gc8b1c92) from inhouse scripts to format the concordance output file.

When the Sequenom report is not present no concordance can be calculated, an empty concordance output file with a header and one row of NAs is created on the fly.

Toolname: BASH programs (sed, awk, uniq, sort, grep, head, cat, echo), plink-1.07, plink 1.08, fastaFromBed (BEDTools-Version-2.11.2), align-vcf-to-ref.pl from inhouse_scripts, iChip_pos_to_interval_list.pl from inhouse_scripts, GATK-1.2-1-g33967a4 UnifiedGenotyper, change_vcf_filter.pl from inhouse scripts,

GATK-1.2-1-g33967a4 VariantEval, extract_info_GATK_variantEval_V3.R (using library from GATK-1.3-24-gc8b1c92) from inhouse scripts.

Scriptname: ConcordanceCheck

Input: SNP file generated with a Sequenom.

Output: concordance file (.concordance.ngsVSSequenom.txt)

# Coverage calculations (Diagnostics only)
### Step 13: Calculate coverage per base and per target

Calculates coverage per base and per target, the output will contain chromosomal position, coverage per base and gene annotation

Toolname: GATK DepthOfCoverage

Scriptname: CoverageCalculations

Input: dedup BAM file (.merged.dedup.bam)

Output: tab delimeted file containing chromosomal position, coverage per base and Gene annotation name (.coveragePerBase.txt)

Per base:
```
java -Xmx10g -jar GenomeAnalysisTK.jar \
	-R human_g1k_v37.fa \
	-T DepthOfCoverage \
	-o region.coveragePerBase \
	--omitLocusTable \
	-I input.bam \
	-L region.bed
```

Per target:
```
java -Xmx10g -jar GenomeAnalysisTK.jar \
	-R human_g1k_v37.fa \
	-T DepthOfCoverage \
	-o region.coveragePerTarget \
	--omitDepthOutputAtEachBase \
	-I input.bam \
	-L region.bed
```

# Metrics calculations
### Step 14 (a,b,c,d): Calculate alignment QC metrics

In this step, QC metrics are calculated for the alignment created in the previous steps. This is done using several QC related Picard tools:

● CollectAlignmentSummaryMetrics
● CollectGcBiasMetrics
● CollectInsertSizeMetrics
● MeanQualityByCycle (machine cycle)
● QualityScoreDistribution
● CalculateHsMetrics (hybrid selection)
● BamIndexStats

These metrics are later used to create tables and graphs (step 24). The Picard tools also output a PDF version of the data themselves, containing graphs.

Toolname: several Picard QC tools

Scriptname: Collect metrics

Input: dedup BAM file (.merged.dedup.bam)

Output: alignmentmetrics, gcbiasmetrics, insertsizemetrics, meanqualitybycycle, qualityscoredistribution, hsmetrics, bamindexstats (text files and matching PDF files)

# Determine Gender
### Step 15: Gender check

Due to the fact a male has only one X chromosome it is important to know if the sample is male or female. Calculating the coverage on the non pseudo autosomal region and compare this to the average coverage on the complete genome predicts male or female well.

Scriptname: GenderCheck
Input: ${dedupBam}.hs_metrics (step 14 & ${dedupBam}.nonAutosomalRegionChrX_hs_metrics (step 10a)

Output: ${sample}.chosenSex.txt

# Variant discovery
### Step 16a: Call variants (VariantCalling)

The GATK HaplotypeCaller estimates the most likely genotypes and allele frequencies in an alignment using a Bayesian likelihood model for every position of the genome regardless of whether a variant was detected at that site or not. This information can later be used in the project based genotyping step.

Toolname: GATK HaplotypeCaller

Scriptname: VariantGVCFCalling

Input: merged BAM files
Output: gVCF file (${sample}.${batchBed}.variant.calls.g.vcf)

```
java -Xmx12g -jar GenomeAnalysisTK.jar \
    -T HaplotypeCaller \
    -R human_g1k_v37.fa \
    -I inputBam.bam \
	-dontUseSoftClippedBases \
	--dbsnp dbsnp_137.b37.vcf \
    -stand_emit_conf 20.0 \
    -stand_call_conf 10.0 \
    -o output.g.vcf \
    -L captured.bed \
    --emitRefConfidence GVCF \
	-ploidy 2  ##ploidy 1 in non autosomal chr X region in male##
```
### Step 16b: Combine variants

When there 200 or more samples the gVCF files should be combined into batches of equal size. (NB: These batches are different then the ${batchBed}.) The batches will be calculated and created in this step. If there are less then 200, this step will automatically be skipped.

Toolname: GATK CombineGVCFs

Scriptname: VariantGVCFCombine

Input: gVCF file (from step 16a)

Output: Multiple combined gVCF files (${project}.${batchBed}.variant.calls.combined.g.vcf{batch}

```
java -Xmx30g -jar GenomeAnalysisTK.jar \
	-T CombineGVCFs \
	-R human_g1k_v37.fa \
	-o batch_output.g.vcf \
	${ArrayWithgVCF[@]}
```
### Step 16c: Genotype variants

In this step there will be a joint analysis over all the samples in the project. This leads to a posterior probability of a variant allele at a site. SNPs and small Indels are written to a VCF file, along with information such as genotype quality, allele frequency, strand bias and read depth for that SNP/Indel.

Toolname: GATK GenotypeGVCFs

Scriptname: VariantGVCFGenotype

Input: gVCF files from step 16a **or** combined gVCF files from step 16b

Output: VCF file for all the samples in the project (${project}.${batchBed}.variant.calls.genotyped.vcf)

```
java -Xmx16g -jar GenomeAnalysisTK.jar \
	-T GenotypeGVCFs \
	-R human_g1k_v37.fa \
	-L captured.bed \
	--dbsnp dbsnp_137.b37.vcf \
	-o output.vcf \
	${ArrayWithgVCF[@]} 
```
### Step 17: Merge batches

Running GATK CatVariants to merge all the files created in the genotype variants step (Step 16c) into one.

Tools: GATK CatVariants

Scriptname: MergeChrAndSplitVariants

Input: files created in step 16 (${project}.{batchBed}.variant.calls.vcf)

Output: merged snp and indel file (${project}.variant.calls.GATK.sorted.vcf)

### Step 18: VariantAnnotator

An HTML file with some statistics and a text file with SNPs per gene and region are produced.

Toolname: GATK VariantAnnotator

Scriptname: StructuralVariantAnnotator

Input: VCF file (.calls.vcf)

Output: SNPeff VCF file (.annotated.vcf)

```
java -Xmx8g -jar GenomeAnalysisTK.jar \
-T VariantAnnotator \
-R human_g1k_v37.fa \
-I input.bam \
-A AlleleBalance \
-A BaseCounts \
-A BaseQualityRankSumTest \
-A ChromosomeCounts \
-A Coverage \
-A FisherStrand \
-A LikelihoodRankSumTest \
-A MappingQualityRankSumTest \
-A MappingQualityZeroBySample \
-A ReadPosRankSumTest \
-A RMSMappingQuality \
-A QualByDepth \
-A VariantType \
-A AlleleBalanceBySample \
-A DepthPerAlleleBySample \
-A SpanningDeletions \
--disable_auto_index_creation_and_locking_when_reading_rods \
-D dbsnp_137.b37.vcf \
--variant input.vcf \
-L captured.bed \
-o output.vcf \
-nt 8
```

### Step 19: Split indels and SNPs

This step is necessary because the filtering of the vcf needs to be done seperately.

Toolname: GATK SelectVariants

Scriptname: SplitIndelsAndSNPs

Input: VCF file (.annotated.vcf)

Output: .annotated.indels.vcf and .annotated.snps.vcf

### Step 20: (a) SNP and (b) Indel filtration

Based on certain quality thresholds (based on GATK best practices) the SNPs and indels are filtered, and marked as Lowqual or Pass.


Toolname: GATK VariantFiltration

Scriptname: VariantFiltration

Input:
- 20a (.snpEff.annotated.snps.vcf)
- 20b (.snpEff.annotated.indels.vcf)

Output:
- 20a filtered snp vcf file (.snpEff.annotated.filtered.snps.vcf)
- 20b filtered indel vcf file (.snpEff.annotated.filtered.indels.vcf)

SNP:
```
java-Xmx8g -Xms6g -jar GenomeAnalysisTK.jar \
-T VariantFiltration \
-R human_g1k_v37.fa\
-o output.vcf \
--variant inputSNP.vcf \
--filterExpression "QD < 2.0" \
--filterName "filterQD" \
--filterExpression "MQ < 25.0" \
--filterName "filterMQ" \
--filterExpression "FS > 60.0" \
--filterName "filterFS" \
--filterExpression "MQRankSum < -12.5" \
--filterName "filterMQRankSum" \
--filterExpression "ReadPosRankSum < -8.0" \
--filterName "filterReadPosRankSum"
```

Indel:
```
java-Xmx8g -Xms6g -jar GenomeAnalysisTK.jar \
-T VariantFiltration \
-R human_g1k_v37.fa\
-o output.vcf \
--variant inputIndel.vcf \
--filterExpression "QD < 2.0" \
--filterName "filterQD" \
--filterExpression "FS > 200.0" \
--filterName "filterFS" \
--filterExpression "ReadPosRankSum < -20.0" \
--filterName "filterReadPosRankSum"
```

### Step 21: Merge indels and SNPs

Merge all the SNPs and indels into one file (per project) and merge SNPs and indels per sample.

Toolname: GATK CombineVariants

Scriptname: MergeIndelsAndSnps

Input: .annotated.filtered.indels.vcf and .annotated.snps.vcf

Output:
- ${sample}.final.vcf
- ${project}.final.vcf

### Step 22: Convert structural variants VCF to table

In this step the indels in VCF format are converted into a tabular format using Perlscript vcf2tab by F. Van Dijk.

Toolname: vcf2tab.pl

Scriptname: IndelVcfToTable

Input: (${sample}.final.vcf)

Output: (${sample}.final.vcf.table)

# QC-ing
### Step 23: In silico concordance check

The reads that are inserted contain SNPs that are handmade. To see whether the pipeline ran correctly at least these SNPs should be found.

Input: InSilicoData.chrNC_001422.1.variant.calls.vcf and ${sample}.variant.calls.GATK.sorted.vcf

Output: inSilicoConcordance.txt

### Step 24a: Prepare QC Report, collecting metrics

Combining all the statistics which are used in the QC report.

Scriptname:QCStats

Toolname: pull_DNA_Seq_Stats.py

Input: metrics files from steps 14a/b/c/d (*.hsmetrics, *.alignmentmetrics, *.insertsizemetrics), flagstat file and concordance file (*.dedup.metrics.concordance.ngsVSarray.txt)

Output: ${sample}.total.qc.metrics.table

### Step 24b: Generate quality control report

The step in the inhouse sequence analysis pipeline is to output the statistics and metrics from each step that produced such data that was collected in the QCStats step before. From these, tables and graphs are produced. Reports are then created and written to a separate quality control (QC) directory, located IN RunNr/Results/qc/statistics. This report will be outputted in html and pdf. Converting html to pdf the tool wkhtmltopdf is used.

Toolname: wkhtmltopdf

Scriptname: QCReport

Input: ${sample}.total.qc.metrics.table

Output: A quality control report html(*_QCReport.html) and pdf (*_QCReport.html)

### Step 25: Check if all files are finished

This step is checking if all the steps in the pipeline are actually finished. It sometimes happens that a job is not submitted to the scheduler. If everything is finished than it will write a file called CountAllFinishedFiles_CORRECT, if not it will make CountAllFinishedFiles_INCORRECT. When it is not all finished it will show in the CountAllFinishedFiles_INCORRECT file which files are not finished yet.

Scriptname: CountAllFinishedFiles

Input: all .sh scripts + all .sh.finished files in the jobs folder

Output: CountAllFinishedFiles_CORRECT or CountAllFinishedFiles_INCORRECT

### Step 26: Prepare data to ship to the customer

In this last step the final results of the inhouse sequence analysis pipeline are gathered and prepared to be shipped to the customer. The pipeline tools and scripts write intermediate results to a temporary directory. From these, a selection is copied to a results directory. This directory has five subdirectories:

o alignment: the merged BAM file with index
o coverage: coverage statistics and plots
o coverage_visualization: coverage BEDfiles
o qc: all QC files, from which the QC report is made
o rawdata/ngs: symbolic links to the raw sequence files and their md5 sum
o snps: all SNP calls in VCF format and in tab-delimited format
o structural_variants: all SVs calls in VCF and in tab-delimited format
Additionally, the results directory contains the final QC report, the worksheet which was the basis for this analysis (see 4.2) and a zipped archive with the data that will be shipped to the client (see: GCC_P0006_Datashipment.docx). The archive is accompanied by an md5 sum and README file explaining the contents.

Scriptname: CopyToResultsDir
# Forensic genomics of a novel _Klebsiella quasipneumoniae_ type from an NICU in China reveals patterns of genetic diversity, evolution and epidemiology

The protocols used to process NGS data generated from 22 clinical isolates of _K. quasipneumoniae_ is provided below. The evolutionary analysis included 4082 publicly available genomes of _Klebsiella_.

* [NGS QC, assembly, and scaffolding][1]
* [Download publicly available _Klebsiella_  genomes][2]
* [Phylogenetics][3]

## Processing Illumina data
Steps to reproduce sequence data processing on a HPC machine with queue system.
### Quality control using [FastQC][4]
- Steps described below are for processing data on a HPC machine with a queue system. Run  `Running_Arrayjobs.pl` to generate individual ‘jobs’ for each isolate. `template_submit.txt` should include the specifications of your HPC and should be within the run folder. This example is for a SLURM system, but can be modified for SGE. A caution to not delete the word PROGRAM.
-  run FastQC for all files `./Running_Arrayjobs.pl -f \*fq -p fastqc -o fastQC.results`
- The FastQC outputs provides quality quality measures. However, problematic samples can be identified using *grep* the word "FAIL" or "WARN" through the summary.txt outputs for all your files.
	`grep "FAIL" \*fastQC.results/summary.txt`
	`grep "WARN" \*fastQC.results/summary.txt`
- If required, trim sequences using [FastX-toolskit][5]. To run in parallel: `./Running_Arrayjobs.pl -p "fastx_trimmer -Q33 -f20 -v -i" -f ../RawData/\*fq -o fast.trimmer.fq`. Modify content between the quotes to specify the details of your trimming.
- Repeat FastQC to check your quality.
### Assembly using [SPAdes][6]
`./RunningSPAdes.pl -f \*fast__trimmer.fq -p n -o SPAdesoutput`
### Generate scaffolds using [Medusa][7]
`java -jar medusa.jar -f ST2727reference/ -i KP1_1.fq_fast__trimmer.fq_SPAdesoutput.fasta -v`
### Organizing scaffoldings using [Mauve][8]
`java -Xmx500m -cp Mauve.jar org.gel.mauve.contigs.ContigOrderer -output KP9_ST23.CP030172.1 -ref  ST23.CP030172.1.fasta -draft KP9_1.fq_fast.trimmer_Velvetoutput.fa`

## Download publicly available _Klebsiella_  genomes
[source][9]
`wget ftp://ftp.ncbi.nlm.nih.gov/genomes/ASSEMBLY_REPORTS/assembly_summary_refseq.txt`
`grep -E "Klebsiella" assembly_summary_refseq.txt | cut -f 8,9,14,15,16 >list`
`grep -E "Klebsiella" assembly_summary_refseq.txt | cut -f 20 >ftp_folder.txt`

`awk 'BEGIN{FS=OFS="/";filesuffix="genomic.fna.gz"}{ftpdir=$0;asm=$10;file=asm"\_"filesuffix;print "wget "ftpdir,file}' ftp_folder.txt > download_fna_files.sh`

`awk 'BEGIN{FS=OFS="/";filesuffix="genomic.gff.gz"}{ftpdir=$0;asm=$10;file=asm"\_"filesuffix;print "wget "ftpdir,file}' ftp_folder.txt > download_gff_files.sh`

`source download_fna_files.sh`

`source download_gff_files.sh`

## Phylogenetic analysis

### Generate core genome alignment using [ROARY][10]
- Prepare the input files for ROARY using: `./ROARYFormat.pl -f FNA/\*.fna -g GFF/\*.gff`
- ROARYFormat.pl concatenates the gff and fna files.
- Run Roary `roary -f KlebsiellaCompleteGenomes -p 16 -e -n  -s -ap --group_limit 200000 -v GFF.FNA.ForROARY/\*gff`

### Generate phylogenies using [RAxML][11]

`raxmlHPC-PTHREADS-SSE3 -T 16 -f a -s KlebsiellaCompleteGenomes/core_gene_alignment.aln  -m GTRGAMMA  -x 12345 -p 12345 -# autoMRE -n KlebsiellaCompleteGenomesPhylo.GAMMA`

### Metadata for all genomes
- Get metadata information from the publicly available genomes using `wget ftp://ftp.ncbi.nlm.nih.gov/biosample/biosample\_set.xml.gz`
- Parsing XML to outbrain Metadata`./XML2Table.pl Biosample.files/\*xml >BioSampleKlebsiella.txt`

### Post Processing tree using R

`source("Function.PruneLargeTRee.r")`
`prune.Large.Tree(tree,keep,drop)`

- **tree**: is a phylogenetic tree; **keep**: a vector with tips to include, even if duplicates exists; **drop**: a vector with tips to ignore
- If variables keep and drop are NULL only the tips with unique branches are kept.
### Generating time-calibrated tree using [LSD][12]
 `./lsd -i KpIIB.LSD.nwk -d Metadata.KpIIB.txt -c -v 1 -f 100 -s 500000`

[1]:	#processing-illumina-data
[2]:	#Download-all-genomes-publicly-available
[3]:	#Phylogenetic-Analysis
[4]:	http://www.bioinformatics.babraham.ac.uk/projects/download.html#fastqc
[5]:	http://hannonlab.cshl.edu/fastx%5C_toolkit/download.html
[6]:	http://cab.spbu.ru/software/spades/
[7]:	https://github.com/combogenomics/medusa
[8]:	http://darlinglab.org/mauve/mauve.html
[9]:	http://www.metagenomics.wiki/tools/fastq/ncbi-ftp-genome-download
[10]:	https://sanger-pathogens.github.io/Roary/
[11]:	https://github.com/stamatak/standard-RAxML
[12]:	http://www.atgc-montpellier.fr/LSD/
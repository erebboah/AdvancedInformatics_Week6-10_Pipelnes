# Advanced Informatics Week 6 Exercises

The goal of this week was to organize raw data using symlinks.

I used R to read in a metadata file (in the case of the RNA-seq data), or simply used the existing paths to make a metadata file indicating the new names for each file.

I looped through the files and used `system(paste0())` to paste the `ln -s` command to the terminal and make symlinks for each file.

The scripts were run as such:
```
Rscript get_symlinks_dna.R
Rscript get_symlinks_atac.R
Rscript get_symlinks_rna.R
```

The scripts output metadata files with the new names for the samples are in the directories `DNAseq`, `ATACseq`, or `RNAseq` with the symlinks in sub-directories named `data`:
```
DNAseq/
    dna_samples.txt
    data/
        ADL06_1_1.fq.gz
        ADL06_1_2.fq.gz
        ADL06_2_1.fq.gz
        ADL06_2_2.fq.gz
        ADL06_3_1.fq.gz
        ADL06_3_2.fq.gz
        ...
ATACseq/
    atac_samples.txt
    data/
        P004_R1.fq.gz
        P004_R2.fq.gz 
        P005_R1.fq.gz
        P005_R2.fq.gz
        P006_R1.fq.gz
        P006_R2.fq.gz
        ...
RNAseq/
    rna_samples.txt
    data/
        x21001B0_R1.fq.gz
        x21001B0_R2.fq.gz
        x21001E0_R1.fq.gz
        x21001E0_R2.fq.gz
        x21001H0_R1.fq.gz
        x21001H0_R2.fq.gz
        ...
```

I also ran `fastqc` on one raw data file from the DNA-seq experiment, `ADL06_1_1`. The results are in the `fastqc` folder and the html file can be viewed [here](http://crick.bio.uci.edu/erebboah/ADL06_1_1_fastqc.html). 

# Advanced Informatics Week 7 Exercises

This week, we mapped the 3 datasets to the reference genome.

On an interactive node, I indexed the reference file here: `ref="/data/class/ecoevo283/erebboah/dmel-all-chromosome-r6.13.fasta"` for BWA, Picard, and HISAT2. 
```
module load bwa/0.7.8
module load samtools/1.10
module load bcftools/1.10.2
module load python/3.8.0
module load gatk/4.1.9.0
module load picard-tools/1.87
module load java/1.8.0
module load hisat2/2.2.1

bwa index $ref
samtools faidx  $ref
java -d64 -Xmx128g -jar /opt/apps/picard-tools/1.87/CreateSequenceDictionary.jar R=$ref O=ref/dmel-all-chromosome-r6.13.dict
hisat2-build $ref $ref
```

I made "prefix" files to use for the bash scripts with the provided code from Dr. Long's notes. For RNA-seq, I made a subsetted prefix file using `scripts/subset_rna.R`, containing 79 samples for the "E" tissue type.
```
ls /data/class/ecoevo283/erebboah/DNAseq/data/*1.fq.gz | sed 's/_1.fq.gz//' > ../prefixes.txt
ls /data/class/ecoevo283/erebboah/ATACseq/data/*R1.fq.gz | sed 's/_R1.fq.gz//' > ../prefixes.txt
ls /data/class/ecoevo283/erebboah/RNAseq/data/*R1.fq.gz | sed 's/_R1.fq.gz//' > ../prefixes.txt
Rscript subset_rna.R
```
The prefix files are in the `DNAseq`, `ATACseq`, and `RNAseq` folders.

The bash scripts that made use of the prefix files are called `align_dna.sh`, `align_atac.sh`, and `align_rna.sh`. DNA-seq data is aligned using BWA followed by generation of read groups using Picard. ATAC-seq data data is simply aligned using BWA. RNA-seq data is aligned using HISAT2 and the resulting `SAM` file is converted to `BAM` and sorted. I ran them with the following commands:
```
sbatch align_dna.sh
sbatch align_atac.sh
sbatch align_rna.sh
```

The scripts folder also contains a brief README about the various R and bash scripts.

The output is in each subfolder named `mapped`:
```
DNAseq/
    dna_samples.txt
    prefixes.txt
    data/
    mapped/
        ADL06_1.RG.bam
        ADL06_1.RG.bam.bai
        ADL06_1.sort.bam
        ADL06_2.RG.bam
        ADL06_2.RG.bam.bai
        ADL06_2.sort.bam
        ADL06_3.RG.bam
        ADL06_3.RG.bam.bai
        ADL06_3.sort.bam
        ...
ATACseq/
    atac_samples.txt
    prefixes.txt
    data/
    mapped/
        P004.sort.bam
        P005.sort.bam
        P006.sort.bam
        ...
RNAseq/
    rna_samples.txt
    prefixes.txt
    prefixes_tissueE.txt
    data/
    mapped/
        x21001E0.sorted.bam
        x21001E0.sorted.bam.bai
        x21002E0.sorted.bam
        x21002E0.sorted.bam.bai
        x21012E0.sorted.bam
        x21012E0.sorted.bam.bai
        ...
```

# Advanced Informatics Week 8 Exercises

The goals for this week were to:
1. work through the DNA-seq analysis pipeline to call SNPs using GATK and generate `VCF` files and 
2. generate counts per gene for RNA-seq data using the subread package.

## DNA-seq GATK pipeline
I made another set of "prefix" files to use for the first step, which is now on a per-sample basis with merged replicates, thus 4 samples: `ADL06`, `ADL09`, `ADL10`, and `ADL14`.
```
ls /data/class/ecoevo283/erebboah/DNAseq/data/*_1_1.fq.gz | sed 's/_1.fq.gz//' > ../prefixes2.txt
```

The prefix file is in the `DNAseq` folder.
I used the code provided in Dr. Long's notes to make 3 bash scripts. I ran the first script to merge sample replicates with Picard `MergeSamFiles`, remove duplicates with GATK `MarkDuplicatesSpark`, and call SNPs with GATK `HaplotypeCaller` to generate one `GVCF` file per sample.
```
sbatch dna_gatk_step1.sh
```

The output is in `DNAseq/gatk`:
```
DNAseq/
    dna_samples.txt
    prefixes.txt
    prefixes2.txt
    data/
    mapped/
    gatk/
        ADL06.dedup.bam
        ADL06.dedup.bam.bai
        ADL06.dedup.bam.sbi
        ADL06.dedup.metrics.txt
        ADL06.g.vcf.gz
        ADL06.g.vcf.gz.tbi
        ...
```

Next, I ran the second script to combine the `GVCF` output per sample into 1 unified file using GATK `CombineGVCFs`:
```
sbatch dna_gatk_step2.sh
```
The output is `DNAseq/gatk/allsample.g.vcf.gz`. (And index `DNAseq/gatk/allsample.g.vcf.gz.tbi`.)

Finally, the third script performs joint genotyping on the combined `GVCF` file to output a final `VCF` using GATK `GenotypeGVCFs`.
```
sbatch dna_gatk_step3.sh
```
The output is `DNAseq/gatk/result.vcf.gz`.

## RNA-seq counts matrix generation
Following the format of the scripts provided for the DNA-seq pipeline, I wrote a bash script to count the number of reads per gene from the 79 sorted `BAM` files using `featureCounts` from the subreads package.
```
sbatch count_rna.sh
```

The output is in `RNAseq/counts`:
```
RNAseq/
    rna_samples.txt
    prefixes.txt
    prefixes_tissueE.txt
    data/
    mapped/
    counts/
        x21001E0.counts.txt
        x21001E0.counts.txt.summary
        x21002E0.counts.txt
        x21002E0.counts.txt.summary
        x21012E0.counts.txt
        x21012E0.counts.txt.summary
        ...
```

For fun, I also concatenated the `*.counts.txt` files into a matrix with each sample as a column and each gene as a row (79 samples x 17,491 genes) using `scripts/make_rna_matrix.R`. The script loops though each sample using the `prefixes_tissueE.txt` file and joins each `.counts.txt` file by the FlyBase gene ID.
```
Rscript make_rna_matrix.R
```
The output is `RNAseq/counts/counts_tissueE_flybaseIDs.tsv`.

For next week, I installed `DESeq2`, `GenomicFeatures`, `Rsamtools`, and `GenomicAlignments` in my R conda enviroment. Some packages installed easily with `install.packages()` and some like `DESeq2` I installed using [conda](https://anaconda.org/bioconda/bioconductor-deseq2), `conda install -c bioconda bioconductor-deseq2`.

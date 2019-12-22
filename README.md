Hap10
======




The goal is to reconstruct accurate and long haplotype from polyploid genome using linked reads (10x genomics reads).


# Workflow of Hap10

Step 0. Preparation procedure

Step 1. Extracting haplotype information

Step 2. Extracting molecule-specific fragments

Step 3. Extracting strongly connected components of fragments

Step 4. Haplotyping



# Usage

First you need to clone the Hap10 package.

```
git clone https://github.com/smajidian/Hap10.git
cd Hap10
```


## Step 0. Preparation procedure

Consider that you have a reference genome, `ref/ref.fa`, and two FASTQ files, `reads/R1.fastq.gz` and `reads/R2.fastq.gz`, corresponding to the Illumina paired-end reads. As mentioned in the paper, you first need to align them to the reference genome using [Longranger](https://support.10xgenomics.com/genome-exome/software/pipelines/latest/installation). Then, variants can be called using [freebayes](https://github.com/ekg/freebayes).

```
longranger mkref ref/ref.fasta
mkdir longranger_output

longranger align --id=longranger_output --fastqs=reads --reference=refdata-ref
freebayes -f ref/ref.fasta -p $k longranger_output/outs/possorted_bam.bam  > var.vcf
```

in which `k` is the ploidy level. The outputs of this step are aligned reads and called SNVs as BAM and VCF files, respectively.


## Step 1. Extracting haplotype information

Here, a fragment file (as a text file) is generated using BAM and filtered VCF file. Firstly, you need to build extract_poly, an edited version (under test) of [extracthairs](https://github.com/vibansal/HapCUT2) in which polyploid genomes are also allowed. The makefile will attempt to build samtools and htslib as git submodules. The output of this step is a binary file `extractHAIRS` in folder `build`.

```
git clone https://github.com/smajidian/extract_poly
cd extract_poly
make
```

The inputs of this step are two files:

- BAM file for an individual containing reads aligned to a reference genome
- VCF file containing only heterzygous SNVs. Complex SNVs and indels are not allowed for this version. The VCF file should only contain genotyped SNPs.

When the SNP rate is high,specially more than 1%, Freebayes has a problem: SNV that are so close to each other are considered as complex variants. `break_vcf.sh` is a bash code from [TriPoly](https://github.com/EhsanMotazedi/TriPoly) to overcome this issue and converts those complex variants to SNVs.

```
utilities/break_vcf.sh  var.vcf var_break.vcf
cat var_break.vcf  | grep -v "1/1/1" | grep -v "0/0/0" grep -v "/2" > var_het.vcf   #This is for triploid.
./extract_poly/build/extractHAIRS --10X 1 --bam out_longranger/outs/possorted_bam.bam --VCF var_het.vcf --out unlinked_fragment_file
```

The second bash line is for triploid, you can easily change it  for higher ploidy levels.


In the output file of the last bash line, `unlinked_fragment_file`,  each line corresponds to one read. Now, We link the reads with the same barcode and generate a barcode-specific fragment file.

```
grep -v "NULL"  unlinked_fragment_file > unlinked_fragment_file_filtered  # removing those reads without barcode
python3 utilities/LinkFragments_brcd_based.py  unlinked_fragment_file_filtered frag.txt
```





## Step 2.  Extracting molecule-specific fragments

The goal of this this step is to extracting molecule-specific fragments from barcode-specific fragment file.

```
python3 utilities/splitter.py frag.txt $m frag_sp.txt
```
in which `m` is the mean 10X molecule length (in Kb) which can be set as 50.


## Step 3.  Extracting strongly connected components of fragments


```
python3 utilities/extract_scc.py frag_sp.txt scc ./out
```
If you want to have larger haplotype block, you can use `cc` instead of `scc`.




## Step 4.  haplotyping (Haplotype assembly core)

For haplotyping, the user can use one of the two modes: fast or accurate. The core of fast mode, Hap++, is SDhaP which is written in C.
The core of accurate mode, Hap10, is MATLAB code.


### Fast mode (Hap++):

You need to install [SDhaP](https://sourceforge.net/projects/sdhap/) using this [instruction](https://github.com/smajidian/sdhapc).

```
python2 utilities/FragmentPoly.py -f frag_sp.txt  -o frag_sd.txt -x SDhaP
./sdhap/hap_poly frag_sd.txt  out.hap 3
python2 utilities/ConvertAllelesSDhaP.py -p out.hap -o out_with_genomic_position.hap -v var_het.vcf  
```



Because, we know that all variants are hetrozygous, we may remove homo variants in estimated haplotype.
```
cat out_with_genomic_position.hap | grep -v "0\t0\t0" |grep -v "1\t1\t1" > out_filtered.hap
```


### Accurate mode (Hap10):

You may refer to this [page](https://github.com/smajidian/Hap10/tree/master/accurate_mode).










## Cite us:

Hap10: reconstructing accurate and long polyploid haplotypes using linked reads. [draft]

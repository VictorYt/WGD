# WGD
**W**hole-**G**enome **D**osage: a suite of tools to evaluate dosage in whole-genome sequencing libraries

**Contact:** Ryan Collins (rcollins@chgr.mgh.harvard.edu)

All code copyright (c) 2016 Ryan Collins and is distributed under terms of the MIT license.  

---  
## Table of Contents  
#### Examples  
- [Example WGD workflow](https://github.com/RCollins13/WGD#example-wgd-workflow)  
- [Pipeline runtimes](https://github.com/RCollins13/WGD#pipeline-runtimes)
- [CNV visualization and annotation](https://github.com/RCollins13/WGD#cnv-visualization-and-annotation)

#### Script documentation  
- [binCov.py](https://github.com/RCollins13/WGD#bincovpy)  
- [WG_binCov.sh](https://github.com/RCollins13/WGD#wg_bincovsh)  
- [makeMatrix.sh](https://github.com/RCollins13/WGD#makematrixsh)  

--- 

### Example WGD workflow  
#### Prerequisites  
The WGD pipeline requires the following:  
- Coordinate-sorted, indexed bams for all samples
- List of contigs to evaluate
- Bed-file of N-masked regions of the reference genome. These are available from [UCSC](http://genome.ucsc.edu/ "UCSC Genome Browser")  

#### Step 1: Generate normalized coverage per chromosome on all libraries  
Normalized coverage is calculated by ```binCov.py``` on a per-chromosome basis. For whole-genome dosage bias analyses, nucleotide coverage at bin sizes of at least 100kb is recommended. Only primary autosomal contigs recommended; e.g. 1...22 for humans. Parallelization of this process is strongly encouraged.  

There are two approaches to parallelization, depending on your available computational resources. Examples are given below using LSF as a scheduler, but could be easily configured to your scheduler/environment.  

**Fully parallelized approach** (almost always preferable)
```
#!/bin/bash  
while read sample; do
  while read contig; do
    bsub "binCov.py ${sample}.bam ${contig} ${sample}.${contig}.rawCov.bed \
    -n ${sample}.${contig}.normCov.bed \
    -b 100000 \
    -m nucleotide \
    -x /path/to/Nmask.bed"
  done < list_of_contigs.txt
done < list_of_samples.txt
```  

Alternatively, if available cores are limited or sample sizes are large, a wrapper script, ```WG_binCov.sh```, will calculate normalized coverage for a set of contigs in serial from a single bam.  

**Semi-parallelized approach** (preferred if available cores are dramatically fewer than [#contigs x #samples])
```
#!/bin/bash  
while read sample; do
  bsub "WG_binCov.sh ${sample}.bam ${sample} `pwd` \
  -L list_of_contigs.txt \
  -b 100000 \
  -m nucleotide \
  -x /path/to/Nmask.bed"
done < list_of_samples.txt
```

#### Step 2: Create normalized dosage matrix for cohort  
Once ```binCov.py``` has completed on all desired contigs for each sample, concatenate the normalized coverage beds per sample into a single bed file. Sorting or ordering of the concatenated bed file is not necessary.  

Creating the coverage matrix can be done easily with ```bedtools unionbedg```, but for sake of convenience has been automated by ```makeMatrix.sh```, as follows:  
```
makeMatrix.sh input.txt > normCov.matrix.bed
```  

An example of the input file (```input.txt``` above):  
```
sample1    /path/to/sample1.cov.bed
sample2    /path/to/sample2.cov.bed
sample3    /path/to/sample3.cov.bed
...
sampleN    /path/to/sampleN.cov.bed
```  

#### Step 3: Run WGD model  
TBD  

#### Step 4 (Optional): Visualization of dosage biases  
TBD  

--- 

### Pipeline runtimes  
**binCov.py**  

The rate-limiting step of the pipeline is ```binCov.py```, which scales linearly with the number of reads per library. A successful run of all 22 human autosomes for a conventional 30X Illumina 2x100bp or 2x150bp WGS library usually takes approximately 3-6 hours (single-core CPU, 2GB RAM, 5GB swap, 1kb bins). This is the reason for maximizing parallelization where possible (see [earlier example](https://github.com/RCollins13/WGD#step-1-generate-normalized-coverage-per-chromosome-on-all-libraries)).  

Physical coverage is marginally faster than nucleotide coverage. Bin sizes ≥100bp do not strongly influence runtime.  

--- 

### CNV visualization and annotation  
#### Companion visualization tool: CNView  
[CNView](https://github.com/RCollins13/CNView) is a companion tool for WGD that can visualize, score, and annotate CNVs directly from ___raw___ ```makeMatrix.sh``` output. More details on CNView can be found [here](http://biorxiv.org/content/early/2016/04/20/049536). If you use CNView, please cite [Collins et al., 2016](http://biorxiv.org/content/early/2016/04/20/049536).

---  
## Script Documentation
---  
### binCov.py
Iterates through a single chromosome of a bam file and calculates either nucleotide or physical coverage in regularly segmented bins.
```
usage: binCov.py [-h] [-n NORM_OUT] [-b BINSIZE] [-m {nucleotide,physical}]
                 [-x BLACKLIST] [-p] [-v OVERLAP] [--oldBT]
                 bam chr cov_out

Calculates non-duplicate primary-aligned binned coverage of a chromosome from
an input BAM file

positional arguments:
  bam                   Input bam
  chr                   Contig to evaluate
  cov_out               Output bed file of raw coverage

optional arguments:
  -h, --help            show this help message and exit
  -n NORM_OUT, --norm_out NORM_OUT
                        Output bed file of normalized coverage
  -b BINSIZE, --binsize BINSIZE
                        Bin size in bp (default: 1000)
  -m {nucleotide,physical}, --mode {nucleotide,physical}
                        Evaluate nucleotide or physical coverage (default:
                        nucleotide)
  -x BLACKLIST, --blacklist BLACKLIST
                        BED file of regions to ignore
  -p, --presubsetted    Boolean flag to indicate if input bam is already
                        subsetted to desired chr
  -v OVERLAP, --overlap OVERLAP
                        Maximum tolerated blacklist overlap before excluding
                        bin
  --oldBT               Boolean flag to indicate if you are using a bedtools
                        version pre-2.24.0
```  
**Usage Notes:**  
- Input bam file must be coordinate-sorted and indexed.  
- Only non-duplicate primary-aligned reads or proper pairs are considered for 'nucleotide' and 'physical' mode, respectively.  
- Normalized coverage is raw coverage per bin divided by median of all non-zero, non-blacklisted bins on the same contig.  
- Bins will be ignored automatically if they share at least ```-v``` percent overlap by size with blacklisted regions (```-x``` or ```--blacklist```).  
- Currently uses ```bedtools coverage``` syntax assuming ```bedtools``` version 2.24.0 or later (i.e. ```-a``` is bins and ```-b``` is reads; this was reversed starting in ```bedtools v2.24.0```). Specify option ```--oldBT``` to revert to bedtools syntax pre-2.24.0.  

---  

### WG_binCov.sh
Wrapper for serialized execution of binCov.py across multiple chromosomes for an individual sample  
```
usage: WG_binCov.sh [-h] [-b BINSIZE] [-m MODE] 
                    [-L CONTIGS] [-x BLACKLIST] [-v OVERLAP] 
                    BAM ID OUTDIR

Wrapper for serialized execution of binCov.py across multiple chromosomes

Positional arguments:
  BAM     Input bam
  ID      Sample ID
  OUTDIR  Output directory

Optional arguments:
  -h  HELP         Show this help message and exit
  -b  BINSIZE      Bin size in bp (default: 1000)
  -m  MODE         Evaluate physical or nucleotide coverage (default: nucleotide)
  -L  CONTIGS      List of contigs to evaluate (default: all contigs in bam header)
  -x  BLACKLIST    BED file of regions to ignore
  -v  OVERLAP      Maximum tolerated blacklist overlap before excluding bin
```  
**Usage Notes:**  
- Contents of arguments are not checked; it's up to the user to ensure they're feeding appropriate files and values.  

---  

### makeMatrix.sh
Helper tool to automate creation of sorted coverage matrices from binCov.py output bed files. Wraps ```bedtools unionbedg```. Note that, in most cases, multiple chromosome outputs from binCov.py from the same sample should be concatenated before being passed to this tool.  
```
usage: makeMatrix.sh [-h] [-o OUTFILE] SAMPLES

Helper tool to automate creation of sorted coverage matrices from
binCov.py output bed files

Positional arguments:
  SAMPLES     List of samples and coverage files (tab-delimmed)

Optional arguments:
  -h  HELP      Show this help message and exit
  -o  OUTFILE   Output file (default: stdout)
```  
**Usage Notes:**  
- Requires ```bedtools``` to be in your path ($PATH environment variable)    

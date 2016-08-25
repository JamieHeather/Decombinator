# innate2adaptive / Decombinator 
## v3.1
##### This version written by James M. Heather, Niclas Thomas, Katharine Best, Theres Oakes, Mazlina Ismail and Benny Chain
##### Innate2Adaptive lab @ University College London, 2016

---

Decombinator is a fast and efficient tool for the analysis of T-cell receptor (TCR) repertoire sequences produced by deep sequencing. The current version (v3) improves upon the previous through:
* incorporation of error-correction strategies using unique molecular index (UMI) barcoding techniques
* optimisation for additional speed and greater accuracy
* usage of extended tag sets of V and J genes allowing detection of genes of all functionalities
* greater command line input, log file production and code commenting, providing greater flexibility, repeatability and ease of modification

## For the original Decombinator paper, please see [Thomas *et al*, Bioinformatics, 2013](http://dx.doi.org/10.1093/bioinformatics/btt004).

<h3 id="top">Navigation:</h3>
* [Installation](#installation)
* [Demultiplexing](#demultiplexing)
* [Decombining](#decombinator)
* [Collapsing](#collapsing)
* [CDR3 extraction](#cdr3)
* [General usage notes](#generalusage)
* [Calling Decombinator from other scripts](#calldcr)

---

TCR repertoire sequencing (TCRseq) offers a powerful means to investigate biological samples to see what sequences are present in what distribution. However current high-throughput sequencing (HTS) technologies can produce large amounts of raw data, which presents a computational burden to analysis. This such raw data will also unavoidably contain errors relative to the original input molecules, which could confound the results of any experiment.

Decombinator addresses the problem of speed by employing a rapid and [highly efficient string matching algorithm](https://figshare.com/articles/Aho_Corasick_String_Matching_Video/771968) to search the FASTQ files produced by HTS machines for rearranged TCR sequence. The central algorithm searches for 'tag' sequences, the presence of which uniquely indicates the inclusion of particular V or J genes in a recombination. If V and J tags are found, Decombinator can then deduce where the ends of the germline V and J gene sections are (i.e. how much nucleotide removal occurred during V(D)J recombination), and what nucleotide sequence (the 'insert sequence') remains between the two. These five pieces of information - the V and J genes used, how many deletions each had and the insert sequene - contain all of the information required to reconstruct the whole TCR nucleotide sequence, in a more readily stored and analysed way. Decombinator therefore rapidly searches through FASTQ files and outputs these five fields into comma delimited output files, one five-part classifier per line.

The Decombinator suite of scripts are written in **Python v2.7**, and the default parameters are set to analyse data as produced by the ligation-mediated 5' RACE TCR amplification pipeline. The pipeline consists of four scripts, which are applied sequentially to the output of the previous starting with TCR-containing FASTQ files (produced using the barcoding 5' RACE protocol):

1. Raw FASTQ files are demultiplexed to individual samples
2. Sample specific files are then searched for rearranged TCRs with Decombinator
3. Decombined data is error-corrected ('collapsed') with reference to the random barcode sequences added prior to amplification
4. Error-corrected data is translated and CDR3 sequences are extracted

---

<h1 id="installation">Installation and getting started</h1>

*Please note that some familiarity with Python and command-line script usage is assumed, but everything should be easily managed by those without strong bioinformatic or computational expertise.*

### Required modules

Python 2.7 is required to run this pipeline, along with the following non-standard modules:

* acora
* biopython
* regex
* python-Levenshtein

These can be installed via pip (although most will likely appear in other package managers), e.g.:
```bash
pip install python-biopython python-levenshtein regex acora
```
If users are unable to install Python and the required modules, a Python installation complete with necessary packages has been bundled into a Docker container, which should be runnable on most setups. The appropriate image is located [on Dockerhub, under the name 'dcrpython'](https://hub.docker.com/r/decombinator/dcrpython/).

### Get scripts

The Decombinator scripts are all downloadable from the [innate2adaptive/Decombinator Github repository](https://github.com/innate2adaptive/Decombinator). These can be easily downloaded in a Unix-style terminal like so:

```bash
git clone https://github.com/innate2adaptive/Decombinator.git
```

Decombinator also requires a number of additional files, which contain information regarding the V and J gene sequences, their tags, and the locations and motifs which define their CDR3 regions. By default Decombinator downloads these files from [the git repository where they are maintained](https://github.com/innate2adaptive/Decombinator-Tags-FASTAs), which obviously requires a working internet connection. In order to run Decombinator offline, these files must be downloaded to a local location, and either stored within the directory where you wish to run Decombinator or specified using the appropriate command line flag. These additional files can also be downloaded via `git clone`:

```bash
git clone https://github.com/innate2adaptive/Decombinator-Tags-FASTAs.git
```

The current version of Decombinator has tag sets for the analysis of alpha/beta and gamma/beta TCR repertoires from both mouse and man. In addition to the original tag set developed for the 2013 Thomas *et al* Bioinformatics paper, an 'extended' tag set has been developed for human alpha/beta repertoires, which covers 'non-functional' V and J genes (ORFs and pseudogenes, according to IMGT nomenclature) as well as just the functional.

### General notes

* All scripts accept gzipped or uncompressed files as input
* By default, all output files will be gz compressed
 * This can be suppressed by using the "don't zip" flag: `-dz`
* All files also output a summary log file
 * We strongly recommend you familarise yourself with them.
* All options for a given script can be accessed by using the help flag `-h`

<sub>[↑Top](#top)</sub>

---

<h1 id="demultiplexing">Demultiplexor.py</h1>
## Demultiplexing: getting sample-specific, barcoded V(D)J containing reads

*This demultiplexing step is designed specifically to make use of the random barcode sequences introduced during the Chain lab's wet lab TCR amplification protocol. While it should be fairly straightforward to adapt this to other UMI-based approaches, that will require some light modification of the scripts. Users wanting to apply Decombinator to demultiplexed, non-barcoded data can skip this step.*

Illumina machines produce 3 FASTQ read files:

* Read 1 
 * Contains the V(D)J sequence and a demultiplexing index (the SP1 index)
* Read 2 
 * Contains the barcode sequences, and reads into the 5' UTR
* Index 1 
 * Contains the second demultiplexing index (the typical Illumina index read, giving the SP2 index)
 
The demultiplexing script extracts the relevant sequences from each read file and combines them into reads in one file, and demultiplexes them on the two indexes to produce separate sample files containing both the V(D)J and barcode information.

Note that you may need to run `bcl2fastq` or alter a configuration file on your sequencing machine in order to obtain the I1 index read file (which is not advised unless you know what you're doing, please consult whoever runs your machines). For example, this can be achieved on a MiSeq by adding the following line to the `Reporter.exe.config`, under `<appsettings>`:

```bash
<add key=”CreateFastqForIndexReads” value=”1”/>
```

You need to provide the demultiplexing script:

* The location of the 3 read files
* An index csv file giving sample names and indexes
 * Sample names will be carried downstream, so use sensible identifiers 
 * Including the chain (e.g. 'alpha') will allow auto-detection in subsequent scripts (if *only one* chain is used per file)
 * Do not use space or '.' characters
* Give numeric indexes, SP1 end first, then SP2 end
 * e.g. `'AlphaSample1,5,9'` would put all reads amplified using SP1-6N-I-5-aRC1 and P7-L-9 in one file

An example command might look like this:
```bash
python Demultiplexor.py -r1 R1.fq.gz -r2 R2.fq.gz -i1 I1.fq.gz -ix IndexFile.ndx
```

It's also likely that your read files are already demultiplexed by the machine, using just the SP2 indexes alone. Single read files can be produced using bash to output all appropriate reads into one file, e.g.:

```bash
# Linux machines
zcat \*R1\* | gzip > R1.fq.gz
# Macs
gunzip -c \*R1\* | gzip > R1.fq.gz
```

If you suspect your indexing might be wrong you can use the `'outputall'` flag (`-a`). This outputs demultiplexed files using all possible combination of indexes used in our protocol, and can be used to identify the actual index combinations contained in a run and locate possible cross-contamination. Note that this is only recommended for troubleshooting and not for standard use as it will decrease the number of successfully demultiplexed reads per samples due to fuzzy index matching. 

Addition of new index sequences will currently require some slight modification of the code, although if people requested the use of a more easily edited external index file that could be incorporated in the next update.

<sub>[↑Top](#top)</sub>

---

<h1 id="decombinator">Decombinator.py</h1>
## Decombining: identifying rearranged TCRs and outputting their 5-part classifiers

This script performs the key functions of the pipeline, as it searches through demultiplexed reads for rearranged TCR sequences. It looks for short 'tag' sequences (using Aho-Corasick string matching): the presence of a tag uniquely identifies a particular V or J gene. If it finds both a V and a J tag (and the read passes various filters), it assigns the read as recombined, and outputs a five-part Decombinator index (or 'DCR'), which uniquely represents a given TCR rearrangement.

All DCR-containing output files are comma-delimited, with the fields of that five part classifier containing, in order:
* The V index (which V gene was used)
* The J index
* Number of V deletions (relative to germ line)
* Number of J deletions
* Insert sequence (the nucleotide sequence between end of deleted V and J)

The V and J indices are arbitrary numbers based on the order of the tag sequences in the relevant tag file (using Python indexing, which starts at 0 rather than 1). Also note that the number of V and J deletions actually just represents how many bases have been removed from the end of that particular germline gene (as given in the germline FASTA files in the additional file repo); it is entirely possible that more bases where actually deleted, and just that the same bases have been re-added. 

Various additional fields may follow the five part classifier, but the DCR will always occupy the first five positions. An example identifier, from a human alpha chain file, might look like this:

```bash
1, 22, 9, 0, CTCTA
```
Which corresponds to a rearrangement between TRAV1-2 (V index **1**, with **9** nucleotides deleted) and TRAJ33 (J index **22**, with **0** deletions), with an insert sequence (i.e. non-templated additions to the V and/or the J gene) of '**CTCTA**'. For beta chains, the insert sequence will contain any residual TRBD nucleotides, although as these genes are very short, homologous and typically highly 'nibbled', they are often impossible to differentiate.

`Decombinator.py` needs to be provided with a FASTQ file, using the `-fq` flag (which, if following the Chain lab protocol will have come from the output of `Demultiplexor.py`). Users can also specify:
* The species: `-sp`
 * 'human' (default) or 'mouse'
* The TCR chain locus: `-c`
 * Just use the first letter, i.e. a/b/g/d
 * Not required if unambiguously specified in filename
 * This flag will override automatic chain detection

```bash
python Decombinator.py -fq AlphaSample1.fq.gz -sp mouse -c a
```

The other important flag is that which specifies the DNA strand orientations that are searched for rearrangements: `-or`, for which users can specify 'forward', 'reverse' or 'both'. As the 5' RACE approach used in the Chain lab means that all recombinations will be read from the constant region, the default is set to reverse.

Decombinator also outputs a number of additional fields by default, in order to facilitate error-correction in subsequent processes. These additional fields are, in order:
* The FASTQ ID (i.e. the first line of four, following the '@' character)
 * This allows users to go back to the original FASTQ file and find whole reads of interest
* The 'inter-tag' sequence, and quality
 * These two fields are the nucleotide sequence and Q scores of the original FASTQ, running from the start of the found V tag to the end of the found J tag 
 * This therefore just lifts the part of the sequence that contains the rearrangement/CDR3 sequences, which will be used for error-correction
* The barcode sequence, and quality
 * The first 30 bases of each read, carried over from R2 during demultiplexing, contains the two random nucleotide sequences which make up the barcode (UMI)
 
```bash
43, 19, 5, 2, TCGACCTC, M01996:15:000000000-AJF5P:1:1101:17407:1648, AAGTGTCAGACTCAGCGGTGTACTTCTGTGCTCTGTCGACCTCAACAGAGATGACAAGATCATCTTTGGAAAAGGGACACG, HHHGHFHHHHG?FGECHHHHHHHHHGHHFHHGGGGFFHHGFGGHHHHHHHHHHHHHHHHHHHHGHHHHHHHHHHHHHGGGH, GTCGTGATCGGCCGGTCGTGATCGTGCACA, 1AA@A?FFF??DFCGGGEG?FGGAGHGFHC
```

By default Decombinator assigns a 'dcr_' prefix and '.n12' file extension to the output files, to aid in downstream command line data handling. However these can can be altered using the `-pf` and `-ex` flags respectively.

As Decombinator uses tags to identify different V/J genes it can only detect those genes that went into the tag set. Both human and mouse have an 'original' tag set, which contains all of the prototypical alleles for each 'functional' TCR gene (as defined by IMGT).
There is also an 'extended' tag set for human alpha/beta genes, which includes tags for the prototypical alleles of all genes, regardless of predicted functionality. This is the default and recommended tag set for humans (due to increased specificity and sensitivity), but users can change this using the `-tg` flag.

Users wanting to run `Decombinator.py` on their own, non-barcoded data should set the flag `-nbc`. Choosing this option will stop Decombinator from outputting the additional fields required for demultiplexing: instead each classifier will simply be appended an abundance, indicating how many times each identifier appeared in that run. Note that this changes the default file extension to '.nbc', which users may wish to change.

The supplementary files required by Decombinator can be downloaded by the script from GitHub as it runs. If running offline, you need to [download the files](https://github.com/innate2adaptive/Decombinator-Tags-FASTAs) to the directory where you run Decombinator, or specifically inform it of the path using the `-tfdir` flag.

Supplementary files are named by the following convention:
[*species*]\_[*tag set*]\_[*gene type*].[*file type*]  
E.g.:
`human_extended_TRAV.tags`

Also please note the TCR gene naming convention, as determined by IMGT:  
TRAV1-2*01  
TR = TCR / A = alpha / V = variable / 1 = family / -2 = subfamily / *01 = allele

<sub>[↑Top](#top)</sub>

---

<h1 id="collapsing">Collapsinator.py</h1>
## Collapsing: using the random barcodes to error- and frequency-correct the repertoire

Assuming that Decombinator has been run on data produced using the Chain lab's (or a comparable) UMI protocol, the next step in the analysis will remove errors and duplicates produced during the amplification and sequencing reactions.

The barcode sequence is contained in one of the additional fields output by `Decombinator.py` in the .n12 files, that which contains the first 30 bases of R2. As Illumina sequencing is particularly error-prone in the reverse read, and that reads can be phased (i.e. they do not always begin with the next nucleotide that follows the sequencing primer) our protocol uses known spacer sequences to border the random barcode bases, so that we can identify the actual random bases. The hexameric barcode locations (N6) are determined in reference to the two spacer sequences like so:
```
I8 (spacer) – N6 – I8 – N6 – 2 base overflow (n)
GTCGTGATNNNNNNGTCGTGATNNNNNNnn
```
The collapsing script can therefore use the spacer sequences to be sure we have found the right barcode sequences.

The `Collapsinator.py` script performs the following procedures:
* Scrolls through each DCR of the input .n12 file
* First performs error-correction (removing TCR reads produced by errors from e.g. PCR)
 * Then looks at all the different DCRs associated with a given barcode
 * Takes the most common inter-tag sequence/DCR combination as the 'true' TCR (as errors are likely to occur during a later PCR cycle, and thus will most often be minority variants, see [Bolotin *et al.*, 2012](http://dx.doi.org/10.1002/eji.201242517)).
 * Compare this against all other DCRs that shared the same barcode
 * Those that only have a few differences are likely erroneous branches, and can be discounted
 * Frequent but significantly different TCRs could represent 'barcode clash', and thus the process repeats
* Second the script estimates the true cDNA frequency 
 * Clusters all barcode sequences found in conjunction with a DCR (within a given threshold) 
 * This is required as there could be errors within the barcode sequences themselves
 * Number of final clusters gives a better estimate of the original frequency of that cDNA molecule
* Outputs a DCR identifier plus an additional sixth field, giving the corrected abundance of that TCR in the sample

Collapsing occurs in a chain-blind manner, and so only the decombined '.n12' file is required, without any chain designation, with the only required parameter being the infile:
```bash
python Collapsinator.py -in dcr_AlphaSample1.n12
```
A number of the filters and thresholds can be altered using different command line flags. In particular, changing the R2 barcode quality score and TCR sequence edit distance thresholds (via the `-mq` `-bm` `-aq` and `-lv` flags) are the most influential parameters. However this the need for such fine tuning will likely be very protocol-speficic, and is only suggested for advanced users, and with careful data validation.

The default file extension is '.freq'.

<sub>[↑Top](#top)</sub>

---

<h1 id="cdr3">CDR3translator.py</h1>
## TCR translation and CDR3 extraction: turning DCR indexes into complementarity determining region 3 sequences

As the hypervariable region and the primary site of antigenic contact, the CDR3 is probably the region of most interest. By convention, the [CDR3 is defined](http://dx.doi.org/10.1016/S0145-305X(02)00039-3) as running from the position of the second conserved cysteine encoded in the 3' of the V gene to the phenylalanine in the conserved 'FGXG' motif in the J gene. However, some genes use non-canonical residues/motifs, and the position of these motifs varies.

In looking for CDR3s, we also find 'non-productive' reads, i.e. those that don't appear to be able to make productive, working TCRs. This is determined based on the presence of stop codons, being out of frame, or lacking appropriate CDR3 motifs. 

The process occurs like so:

```
# Starting with a Decombinator index
43, 5, 1, 7, AGGCAGGGATC

# Used to construct whole nucleotide sequences, using the germline FASTAs as references
GATACTGGAGTCTCCCAGAACCCCAGACACAAGATCACAAAGAGGGGACAGAATGTAACTTTCAGGTGTGATCCAATTTCTGAACACAACCGCCTTTATTGGTACCGACAGACCCTGGGGCAGGGCCCAGAGTTTCTGACTTACTTCCAGAATGAAGCTCAACTAGAAAAATCAAGGCTGCTCAGTGATCGGTTCTCTGCAGAGAGGCCTAAGGGATCTTTCTCCACCTTGGAGATCCAGCGCACAGAGCAGGGGGACTCGGCCATGTATCTCTGTGCCAGCAGCTTAGAGGCAGGGATCAATTCACCCCTCCACTTTGGGAATGGGACCAGGCTCACTGTGACAG

# This is then translated into protein sequence
DTGVSQNPRHKITKRGQNVTFRCDPISEHNRLYWYRQTLGQGPEFLTYFQNEAQLEKSRLLSDRFSAERPKGSFSTLEIQRTEQGDSAMYLCASSLEAGINSPLHFGNGTRLTVT

# The CDR3 sequence is then extracted based on the conserved C and FGXG motifs (as stored in the .translate supplementary files)
CASSLEAGINSPLHF
```

In order to do so, a third kind of supplementary data file is used, .translate files, which provide the additional information required for CDR3 extraction for each gene type (a/b/g/d, V/J). They meet the same naming conventions as the tag and FASTA files, and consist of four comma-delimited fields, detailing:
* Gene name
* Conserved motif position (whether C or FGXG)
* Conserved motif sequence (to account for the the non-canonical)
* IMGT-defined gene functionality (F/ORF/P)

`CDR3translator.py` only requires an input file, and to be told which TCR chain is being examined (although this can also be inferred from the file name, as with `Decombinator.py`. Input can be collapsed or not (e.g. .freq or .n12), and the program will retain the frequency information if present.

```bash
python CDR3translator.py -in dcr_AlphaSample1.freq
# or
python CDR3translator.py -in dcr_AnotherSample1.freq -c b
```

Outputs a '.cdr3' file by default, which looks like:
`CASSISGRLDTQYF, 1`

Alternatively you can use the 'dcr output' flag (`-do`) to output DCRs alongside the CDR3s they encode, e.g.:
`18, 8, 5, 5, ATCAGCGGGAGATT:CASSISGRLDTQYF, 1`

You can also use the 'nonproductive' flag  (`-np`) to output non-productive rearrangements to a separate file:
`15, 3, 1, 5, CTGTATCAGGGGGCC:OOF_with_stop, 1`

The residues that make up the 'GXG' section of the 'FGXG' motif can also be included if desired, through use of the `-gxg` flag.

<sub>[↑Top](#top)</sub>

---

<h1 id="generalusage">General usage notes and tips</h1>

## Being efficient
* Use the help flag (`-h`) to see all options for a given script
* Where possible, run scripts in same folder as data
 * Should work remotely, but recommended not to
* Use bash loops to iterate over many samples
 * For example, to run the CDR3 translation script on all alpha .freq files in the current directory:
`for x in *alpha*.freq.gz; do echo $x; python ligTCRtranslateCDR3.py -in $x -c a; done`
* There is also a [makefile in development](https://github.com/innate2adaptive/automate-decombinator) to automate the pipeline from start to finish
* Ideally don't pool alpha and beta of one individual into same index for sequencing
 * You may wish to know the fraction of reads Decombining, which will be confounded if there are multiple samples or chains per index combination

## Be aware of the defaults
* These scripts are all set up to analyse data produced from our ligation TCR protocol, and thus the defaults reflect this
 * By far the most common libraries analysed to date are human αβ repertoires
* If running on mouse and/or gamma/delta repertoires, it may be worth altering the defaults towards the top of the code to save on having to repeatedly have to set the defaults
* If running on data not produced using our protocol, likely only the Decombinator and translation scripts will be useful 
  * The `-nbc` (non-barcoding) flag will likely need to be set (unless you perform analogous UMI-positioning to `Demultiplexor.py`)
  * If your libraries contain TCRs in the forward orientation, or in both, this will also need to set with the `-or` flag

## Keep track of what's going on
* Using the 'dontgzip' flag ('-dz') will stop the scripts automatically compressing the output data, which will minimise processing time (but take more space)
 * It is recommend to keep gzipped options as default
 * Compressing/decompressing data slightly slows your analysis, but dramatically reduces the data storage footprint
* Don't be afraid to look at the code inside the scripts to figure out what's going on
* Check your log files, as they contain important information
* **Back up your data** (including your log files)!

## Use good naming conventions
* Keep separate alpha and beta files, named appropriately
  * Keep chain type in name
  * Decombinator will always take specified chain as the actual, but failing that will look in filename
* Don't use full stops in file names
 * Some of the scripts use them to determine file extension locations
 * Additionally discourages inadvertent data changing
* While there are default file extensions, these settings can be set via the `-ex` flag 
 * Similarly, when Decombining you can set the prefix with `-pf`
 
## Using Docker
* Docker may be convenient for systems where installing many different packages is problematic
 * It should allow users to run Decombinator scripts without installing Python + packages (although you still have to install Docker)
 * The scripts however will run faster if you natively install Python on your OS
* If opted for, users must first [install Docker](https://docs.docker.com/engine/installation/)
* Then scripts can be run like so:
```bash
docker run -it --rm --name dcr -v "$PWD":/usr/src/myapp -w /usr/src/myapp decombinator/dcrpython python SCRIPT.py`
```
* Depending on your system, you may need to run this as a superuser, i.e. begin the command `sudo docker...`
* N.B.: the first time this is run Docker will download [the Decombinator image from Dockerhub](https://hub.docker.com/r/decombinator/dcrpython/), which will make it take longer than usual

<sup>[↑Top](#top)</sup>

---

<h1 id="calldcr">Calling Decombinator from other Python scripts</h1>

As an open source Python script, `Decombinator` (and associated functions) can be called by other scripts, allowing advanced users to incorporate our rapid TCR assignation into their own scripts, permitting the development of highly bespoke functionalities.

If you would like to call Decombinator from other scripts you need to make sure you fulfil the correct requirements upstream of actually looking for rearrangements. You will thus need to:
* Set up some 'dummy' input arguments, as there are some that Decombinator will expect
* Run the 'import_tcr_info' function, which sets up your environment with all of the required variables (by reading in the tags and FASTQs, building the tries etc)

An example script which calls Decombinator might therefore look like this:

```python
import Decombinator as dcr

# Set up dummy command line arguments 
inputargs = {"chain":"b", "species":"human", "tags":"extended", "tagfastadir":"no", "lenthreshold":130, "allowNs":False, "fastq":"blank"}

# Initalise variables
dcr.import_tcr_info(inputargs)

testseq = "CACTCTGAAGATCCAGCGCACACAGCAGGAGGACTCCGCCGTGTATCTCTGTGCCAGCAGCTTATTAGTGTTAGCGAGCTCCTACAATGAGCAGTTCTTCGGGCCAGGGACACGGCTCACCGTGCTAGAGGACCTGAAA"

# Try and Decombine this test sequence
print dcr.dcr(testseq, inputargs)
```

This script produces the following list output:
`[42, 6, 2, 0, 'TTAGTGTTAGCGAG', 22, 118]`

The first five fields of this list correspond to the standard DCR index as described above, while the last two indicate the start position of the V tag and the end position of the J tag (thus definining the 'inter-tag region').

Users may wish to therefore loop through their data that requires Decombining, and call this function on each iteration. However bear in mind that run in this manner Decombinator only checks whatever DNA orientation it's given: if you need to check both orientations then I suggest running one orientation through first, checking whether there's output, and if not then running the reverse orientation through again (which can be easily obtained using the Seq functionality of the Biopython package).

<sup>[↑Top](#top)</sup>

---

### Legacy Decombinator versions

If users wish to view previous versions of Decombinator, v2.2 is available from the old GitHub repo [uclinfectionimmunity / Decombinator](https://github.com/uclinfectionimmunity/Decombinator/). However it is not recommended that this script be used for analysis as it lacks some key error-reduction features that have been integrated into subsequent versions, and is no longer supported.

---

### Related reading

* [Thomas et al (2013), Bioinformatics, “Decombinator: a tool for fast, efficient gene assignment in T-cell receptor sequences using a finite state machine”](http://dx.doi.org/10.1093/bioinformatics/btt004)
 * The original Decombinator paper, which explains the principles of the script 
* [Heather et al (2016), Frontiers in Immunology, “Dynamic Perturbations of the T-Cell Receptor Repertoire in Chronic HIV Infection and following Antiretroviral Therapy”](http://dx.doi.org/10.3389/fimmu.2015.00644)
 * Paper describing the previous version of the protocol and some applications (as applied to studying the repertoire of HIV+ patients), which also introduces the collapsing procedure
* [Best et al (2015), Scientific Reports, “Computational analysis of stochastic heterogeneity in PCR amplification efficiency revealed by single molecule barcoding”](http://dx.doi.org/10.1038/srep14629)
 * An in-depth analysis of barcoded data showing the need for barcoding in producing robustly quantitative data from the stochastic maelstrom of amplification that makes up PCR



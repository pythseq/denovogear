# Rough Draft Instructions for using denovogear 
Authors: Don Conrad, Avinash Ramu ( aramu at genetics dot wustl dot edu ) and Reed Cartwright

## RELEASE NOTES
v0.5.2
Added read-depth, posterior-probability filters.
Output number of sites in the BCF and number of sites passing filters.
Modified paired caller output.

v0.5.1
Fixed bug in triallelic configuration. Some trialleic denovo configurations were being called incorrectly.

## COMPILING 
Compilation of DeNovoGear requires CMake.  You can download CMake installers from the CMake
website <http://www.cmake.org/cmake/resources/software.html>.  Most Linux
distributions allow you to install CMake using their package software.

Compiling and Installing on Linux:
        `tar xvzf denovogear*.tar.gz`
        `cd denovogear*/build`
        `cmake ..`
        `make`
        `sudo make install`

Creating Packages:
        `make package`
        `make package_source`

## BINARIES
Binaries are available for MacOSX and for Linux.

## RUNNING THE CODE

### Finding Denovo Mutations in trios and pairs.

DeNovoGear takes in a PED file and a BCF file as input. The PED file describes the relationship between the samples and the BCF file contains the sequencing information for every locus.

#### usage:
	     `./denovogear auto dnm --ped sample.ped --bcf sample.bcf`

#####  about sample.bcf:
BCF files can be generated from the alignment using the samtools mpileup 
command. The command to generate a bcf file from sample.bam is:
	`samtools mpileup -gDf reference.fa sample.bam > sample.bcf`

The -D option of the samtools mpileup command retains the per-sample read depth 
which is preferred by denovogear as it helps to filter out sites without a minimum number of reads(but note that DNG will work without per-sample RD information, in which case the RD tag encodes the average read depth information). The -g option computes genotype likelihoods and produces a compressed bcf output and the -f option is used to indicate the reference fasta file against which the alignment was built. A sample BCF file 'sample_CEU.bcf' is included in the distribution.

#####  about sample.ped: 
The PED file contains information about the trios present in the BCF file. 
Please make sure that all the members of the trios specified in the PED file 
are present in the BCF file. The PED file can be used to specify a subset of 
individuals for analysis from the BCF (in other words not every sample in the BCF needs to 
be represented in the PED file).

The PED file is a tab delimited file. The first six columns of the PED file are 
mandatory, these are Family ID, Individual ID, Paternal ID, Maternal ID, 
Sex (1 = male; 2 = female; other = unknown) and Phenotype. Denovogear makes use of the first four columns. The sample ID's in the PED file need to be exactly as they appear in the BCF file header. Sample order within the PED file does not matter, as family relationships are completely specified by the value of the child/mother/father fields in each row.
 
For example, a single line in the PED file that specifies a trio looks like:

CEU	NA12878_vald-sorted.bam.bam	NA12891_vald-sorted.bam.bam	NA12892_vald-sorted.bam.bam	2	2

An example PED file, sample_CEU.ped, is included in the distribution directory. 

##### about "snp_lookup.txt" and "indel_lookup.txt": 
These are tables with precomputed priors (and other useful numbers) for all possible 
trio configurations, under the null (no mutation present) and alternative (true de novo). 
The default tables are generated during each program run using a prior of 
1 x 10 ^-8 /bp/generation on the haploid germline point mutation rate, 
and 1 x 10 ^-9 /bp/generation on the haploid germline indel mutation rate. 

If you wish to change the default point or indel mutation rates use the --snp_mrate 
or --indel_mrate switches respectively. 

For example
	     `./denovogear auto dnm --ped sample.ped --bcf sample.bcf --snp_mrate 2e-10 --indel_mrate 1e-11`

The indel mutation rate prior is calculated based on the length of the insertion or deletion event, 
separate models are used for insertions and deletions. The two models are based on the indel observations from the 1000Genomes phase 1 data.

The insertion mutation rate is modeled using the function
 	log (mrate) = mu_scale * (-22.8689 - (0.2994 * insertionLength))

The deletion mutation rate is modeled using the function
	log (mrate) = mu_scale * (-21.9313 - (0.2856 * deletionLength))

Note that a constant factor is used to scale the mutation rate, it is set to 1.0 
by default and can be set using the switch --mu_scale. This provides the users to scale the mutation rate prior according to their data-set.

For example, 
	     `./denovogear auto dnm --ped sample.ped --bcf sample.bcf --mu_scale 3`


#### OUTPUT FORMAT for TRIOS

The output format is a single row for each putative de novo mutation (DNM), with the following fields

        1. Event type (POINT MUTATION or INDEL).
        2. CHILD_ID - sample ID of trio-offspring with the DNM.
        3. ref_name - chromosome. 
        4. coor - physical Position. 
        5. ref - base present in reference sequence at this position (from BCF).
        6. alt - comma separated list of alternate non-reference alleles called on at-least one sample (from BCF).
        7. maxlike_null  - likelihood of the most likely mendelian-compatible genotype configuration.
        8. pp_null - posterior probability of the most likely mendelian-compatible genotype configuration.        
        9. tgt_null(child/mom/dad)  - genotypes of the most likely mendelian-compatible configuration.
        10. snpcode - code that indicates whether the configuration shown in field 6 is monomorphic (1) or contains variation (2)(internal filters, can be ignored).
        11. code - This field seems to be redundant to field 7, except the codes are (6) and (9).(internal filters, can be ignored).
        12. maxlike_dnm - likelihood of the most likely DENOVO genotype configuration.
        13. pp_dnm - posterior probability of the most likely DENOVO genotype configuration.
        14. tgt_dnm(child/mom/dad)  - genotypes of the most likely mendelian-compatible configuration.
        15. lookup - Code that indicates if the most likely DNM is a transition (4) or transversion (5) (for development use only).
        16. flag - Flag that indicates whether the data for the site passed internal QC thresholds (for development use only).
        17-19. Read depth of child, parent 1 and parent 2. 
        20-22. Root mean square of the mapping qualities of all reads mapping to the site for child, parent 1 and parent 2. Currently these values are the same for all samples when using BCF as the input format.

Fields 17-22 are meant for filtering out low quality sites. 

### Separate models for the X chromosome

Denovogear has separate models for autosomes, X chromosome in male offspring and X chromosome in female offspring, 

#### Autosomes model usage

        `./denovogear dnm auto --ped paired.ped --bcf sample.bcf`

#### X in male offspring model usage 

        `./denovogear dnm XS --ped paired.ped --bcf sample.bcf`

#### X in female offspring model usage 

        `./denovogear dnm XD --ped paired.ped --bcf sample.bcf`

### PAIRED SAMPLE ANALYSIS 
DNG can be used to analyze paired samples, it is run the same way as for trios the only difference being the way samples are specified in the PED file,

#### Usage:
 
        `./denovogear dnm auto --ped paired.ped --bcf sample.bcf`

About the arguments, 
	
	1. paired.ped is a ped file containing the family-name and the name of the two samples in the bcf file. The last three columns are mandated by the 	PED format but are ignored by the program. An example line looks like
		F150    NA19240_blood_vald-sorted.bam.bam NA19240_vald-sorted.bam.bam   0       0       0
	2. sample.bcf is a bcf file containing both the samples. 

A sample PED file sample_paired.ped for paired sample analysis is provided with the package.

### PHASER 
DNG can be used to obtain parental phasing information for Denovo Mutations where phase informative sites are present. This is done by looking at reads which cover both the denovo base and a phase informative positions. Phase informative positions lie within a certain window from the denovo site, the default value is 1000 but it can be set by the user.

#### Usage:

        `./denovogear phaser --dnm dnms_file --pgt gts_file --bam alignment --window NUM[1000]`

About the arguments, 

	1. 	dnms_file is the list of DNMs whose parental origin is to be determined. It is a tab delimited file of the format
	  	chr pos inherited_base mutant_base
	2.	gts_file contains the genotypes of the parents and the child. It is a tab delimited file of the format
		chr pos parent1_GT parent2_GT child_GT
	3. 	The third argument is the alignment file (.bam) containing the reads covering the DNM. 
	4. 	Window size is an optional argument which is the maximum distance between the DNM and a phasing site. The default value is 1000. 

#### Output
	DNM_pos 1:182974758     INHERITED G     VARIANT A
        HAP POS 182974328 p1: CC p2: TC Number of denovo-phasing pairs found: 0
        HAP POS 182974572 p1: CC p2: TC Number of denovo-phasing pairs found: 2
                Base at DNM position: A Base at phasing position: C      INFERRED PARENT OF ORIGIN for DNM: p1 SUPPORTING READ COUNT: 3
                Base at DNM position: G Base at phasing position: T      INFERRED PARENT OF ORIGIN for DNM: p1 SUPPORTING READ COUNT: 3
        HAP POS 182974598 p1: GG p2: CG Number of denovo-phasing pairs found: 2
                Base at DNM position: A Base at phasing position: G      INFERRED PARENT OF ORIGIN for DNM: p1 SUPPORTING READ COUNT: 1
                Base at DNM position: G Base at phasing position: C      INFERRED PARENT OF ORIGIN for DNM: p1 SUPPORTING READ COUNT: 2
        HAP POS 182974602 p1: TT p2: CT Number of denovo-phasing pairs found: 2
                Base at DNM position: A Base at phasing position: T      INFERRED PARENT OF ORIGIN for DNM: p1 SUPPORTING READ COUNT: 1
                Base at DNM position: G Base at phasing position: C      INFERRED PARENT OF ORIGIN for DNM: p1 SUPPORTING READ COUNT: 1
        HAP POS 182974707 p1: TT p2: CT Number of denovo-phasing pairs found: 2
                Base at DNM position: A Base at phasing position: T      INFERRED PARENT OF ORIGIN for DNM: p1 SUPPORTING READ COUNT: 25
                Base at DNM position: G Base at phasing position: C      INFERRED PARENT OF ORIGIN for DNM: p1 SUPPORTING READ COUNT: 24
        HAP POS 182974750 p1: AA p2: TA Number of denovo-phasing pairs found: 2
                Base at DNM position: A Base at phasing position: A      INFERRED PARENT OF ORIGIN for DNM: p1 SUPPORTING READ COUNT: 45
                Base at DNM position: G Base at phasing position: T      INFERRED PARENT OF ORIGIN for DNM: p1 SUPPORTING READ COUNT: 47


	For each DNM, a list of of eligible phasing sites from the genotypes file is obtained, these are the loci which lie within the phasing window and which do not have a het/het GT configuration for the parents and where the child is het. The number of DNM and phasing  allele combinations seen in the read level data is output by the program. For the phasing sites, the inferred parental origin is displayed.

For example if the base at the phasing site is T and the parental genotypes are P1:CC and p2:TC at this site, then the parent of origin of this base is p2. By looking at the base in the denovo position on this read it is possible to infer the parent of origin of the denovo mutation. 

A sample list of dnms file, sample_phasing_dnm_f is provided. Also a sample genotypes file sample_phasing_GTs_f is provided for reference.

Please feel free to contact the authors about any concerns/comments.	

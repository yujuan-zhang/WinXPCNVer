# WinXPCNVer

## What it does

Compare probe intensities between two populations and find windows with high Vst values. Start with the two bundled intensity tables.

## Input

Input: the two bundled tab-separated intensity tables; marker IDs and order must match.

## Output

Output: `testpop_refpop.Vst.txt` (2,740 data rows, five columns) and `testpop_refpop.Vst.VstW.win3000.5prb.top3.txt` (126 data rows, seven columns). Headers are not counted. Both steps completed in about 1.1 seconds on macOS, using bundled examples and system Perl.

## Try it

### Verified bundled example

From the repository root, use Perl directly so executable permissions and PATH setup are unnecessary:

```bash
perl calcVst.pl -t testpop.locus_summary -r refpop.locus_summary
perl WinXPCNVdiffer.pl -v testpop_refpop.Vst.txt --winsize 3000 --prb_cri 5 --prb_top 3
```

Reference outputs are in `examples/expected/`. For this unchanged example, compare:

```bash
diff examples/expected/testpop_refpop.Vst.txt testpop_refpop.Vst.txt
diff examples/expected/testpop_refpop.Vst.VstW.win3000.5prb.top3.txt testpop_refpop.Vst.VstW.win3000.5prb.top3.txt
```

No diff output means an exact match. Reference files are functional regression examples, not evidence of a biological signal. Running the scripts again overwrites the correspondingly named output files.

WinXPCNVer is tool package for calculating the Vst values between two populations probe by probe in a sliding window, which could be used to detect highly differentiated variants between populations.

### Overview

**WinXPCNVer** is tool package for calculating the ***Vst*** values between two populations probe by probe in a sliding window, which could be used to detect highly differentiated variants between populations.

Generally, there are 2 steps if there is no significant batch effect for test (reference) population:

1. Calculate *Vst* for each probe  
```
calcVst.pl -t pop1.locus_summary -r pop2.locus_summary
```

2. Calculate *Vst-w* (the averaged Vst of the top N probes in the sliding window of length L(bp) with at least K probes)  
```
WinXPCNVdiffer.pl -v pop1_pop2.Vst.txt --winsize L --prb_cri K --prb_top N
```

The results can be ranked according to the averaged *Vst-w*, the window with high *Vst-w* is likely to be the differentiated variant between populations.  
Please interpret the results with caution**!** As the microarray data could be very noisy, **manually-checking** the probe intensities is required and a comprehensive assessment of the signal (e.g. whether the region has been reported with CNV before, or functional annotation) is suggested.

If there is more than one batch in test (reference) population, we suggest perforing an additional step after Step1 to reduce the batch effect. This step is not required but highly suggested especially when raw data of the microarray were generated in different labs or at different time.  
To illustrate this idea, we assume the test populations from two batches (named *TEST1* and *TEST2*), and the reference population (*REF*) from one batch.  
We can calculate *Vst'* as following:

>Vst'=Vst[TEST1_vs_REF]+Vst[TEST2_vs_REF]-Vst[TEST1_vs_TEST2]

In this case, we have a script **Vst_prime_3pop.pl** to calculate the above formular (See __Vst_prime_3pop.pl__ for detail).  
The basic idea is to **increase the true difference of the variance between populations and reduce the noise caused by batch effect**.  
As an extension, a test population contains i batches and a reference population contains j batches.  
The *Vst'* can be calculated as following:

>Vst'=SUM(pairwise_Vst[TEST_vs_REF])-SUM(pairwise_Vst[TEST_vs_TEST])-SUM(pairwise_Vst[REF_vs_REF])

We provide a simple calculator **Vst_prime_calculator.pl** to add or minus *Vst* values from two Vst files.  
After this additional step, we can use the *Vst'* as input for calculating *Vst-w*.

---
###### Tool Usage

**calcVst.pl** is used to calculate basic _Vst_ value for each marker/probeset between two populations (denoted as test population and reference population) in the script.  
The typical input files are the '**.locus_summary**' from ***Birdsuite***[1] results, however, it is not restricted to the *Birdsuite* output. Any files following the format could be used as input:  
The first four columns are unique markerid/probesetid, chromosome, position, and probe type (not important for the calculation), followed by the individual intensity values.  
For example:

|marker_id |chrom |pos |type |sample1 |sample2 |sample3 |  
|:---------|:-----|:---|:----|:-------|:-------|:-------|
|CN_134 |3 |7689933 |C |1111.34 |1345.95 |976.53 |
|rs1234 |5 |3234455 |S |9999 |7777 |8888 |

Please note that the header 'marker_id chrom pos type sample1 sample2 sample3 …' is **required** in the input file.  
The leading annotation lines starting with '#' will be automatically ignored.  
The markers/probesets and their order should be exactly the same in the two input files.

====================**calcVst.pl** USAGE====================

calcVst.pl

--tpop_lsum|-t test.pop.locus_summary
			the locus_summary of the test population
		--rpop_lsum|-r reference.pop.locus_summary
			the locus_summary of the reference population

--help|-h
			print help document

# the relationship of the test and reference populations here is relative,  
    this only determines the name of the output file (i.e., test.pop_reference.pop.Vst.txt ).

============================================================

The output file is the *Vst* values for markers/probesets. The first four columns are the same as that of the input *.locus_summary* files. The fifth column is the *Vst* value. Note that we now only calculate for autosomes.

**Vst_prime_calculator.pl** is a simple calculator (supporting operations only '+' and '-') for two *Vst* files.  
The format of the input *Vst* file is the same as the output of the **calcVst.pl** (namely, markerid/probesetid -> chrom -> position -> probe_type -> Vst).  
Note that the markers/probesets and their order should be exactly the same in the two input files.

====================**Vst_prime_calculator.pl** USAGE====================

Vst_prime_calculator.pl

--vst_1|-1 vst_1.txt
			the first file of vst between two populations
		--vst_2|-2 vst_2.txt
			the second file of vst between two populations
		--op|o {a|add|m|minus|+|-}
			specify the operation to apply to the two vst files. only 'add', 'a', 'minus',  
            'm', '+', or '-' are allowed.

--help|-h
			print help document

# Algorithm:  Vst' = Vst_1 +/- Vst_2

============================================================

The output file is the so-called *Vst'*. The file format is also the same as the output of **calcVst.pl**.

**Vst_prime_3pop.pl** is a toy script to perform the method to reduce the noise for the same population genotyped from different batches, which is described in our paper, based on three *Vst* files.  
Note that the markers/probesets and their order should be exactly the same in the input files.

====================**Vst_prime_3pop.pl** USAGE====================

Vst_prime_3pop.pl

--vst_1|-1 rpop_t1pop.vst.txt
			the vst between ref population and test1 population (namely, batch.1)
		--vst_2|-2 rpop_t2pop.vst.txt
			the vst between ref population and test2 population (namely, batch.2)
		--vst_3|-3 t1pop_t2pop.vst.txt
			the vst between test1 population (namely, batch.1) and test2 population (namely,  
			batch.2)

--help|-h
			print help document

# Algorithm:  Vst' = Vst_r_t1 + Vst_r_t2 - Vst_t1_t2

============================================================

**WinXPCNVdiffer.pl** performs a window-based scanning on *Vst* signals. Given a number of markers/probesets as a criterion (*`--prb_cri`*), if a window includes markers/probesets satisfying the criterion, the top n (*`--prb_top`*) *Vst* values are averaged as the *Vst-w* value for this window.  
Note that the input file (*Vst* or *Vst'*) should be sorted first by chromosome, and then by position.

====================**WinXPCNVdiffer.pl** USAGE====================

WinXPCNVdiffer.pl

--vst_prime|-v vst_prime.txt or Vst.txt
			the vst_prime or Vst results (from 'Vst_prime_calculator.pl', or 'calcVst.pl',  
            respectively)

--winsize integer
			size of the sliding window, in basepair, 1000 as default.
		--prb_cri integer
			the criterion of the number of markers/probesets in a window, 3 as default.
		--prb_top integer
			the number of the top Vst values in a window to calculate the Vst-w statistics.
		--overlap number
			the overlap of sliding windows, could be less than 1 (means proportion of the  
            winsize) or be an integer (means in basepairs), automatically detected.  
            0 as default.

--help|-h
			print help document

============================================================

The output includes 7 columns, with one window per line. The first three columns give the region of the window (i.e., chromosome, boundary marker/probeset positions for start and end), the fourth column is the number of the markers/probesets in this window, followed by the **mean** and **sd** of the top n *Vst* values in this window (i.e., ***Vst-w*** statistics), and the last column gives the markerids/probesetids.  
Note that we now only calculate for autosomes.

---

###### Illustration of the filename style

We give a simple example to illustrate the output of the filenames of our scripts.  
For example, we have two '**.locus_summary**' files: '*pop1.locus_summary*' and '*pop2.locus_summary*'. Then we run commands (in '`>`' lines) and give the *NEW* output files (in '`#`' lines), supposing users in *Linux/Unix* environment:

>ls 
	# pop1.locus_summary pop2.locus_summary

>calcVst.pl -t pop1.locus_summary -r pop2.locus_summary
	# pop1_pop2.Vst.txt

>calcVst.pl -t pop2.locus_summary -r pop1.locus_summary
	# pop2_pop1.Vst.txt

>Vst_prime_calculator.pl -1 pop1_pop2.Vst.txt -2 pop2_pop1.Vst.txt -o '+' 
	# pop1_pop2.Vst.pop2_pop1.Vst.add.Vst_prime.txt

>Vst_prime_calculator.pl -1 pop1_pop2.Vst.txt -2 pop2_pop1.Vst.txt -o m
	# pop1_pop2.Vst.pop2_pop1.Vst.minus.Vst_prime.txt

>WinXPCNVdiffer.pl -v pop1_pop2.Vst.txt --winsize 3000 --prb_cri 5 --prb_top 3 
	# pop1_pop2.Vst.VstW.win3000.5prb.top3.txt

### Citation

When using ```WinXPCNVer```, please cite:

Lou H, Lu Y, Lu D, Fu R, Wang X, Feng Q, Wu S, Yang Y, Li S, Kang L, Guan Y, Hoh BP, Chung YJ, Jin L, Su B, Xu S. A 3.4-kb Copy-Number Deletion near EPAS1 Is Significantly Enriched in High-Altitude Tibetans but Absent from the Denisovan Sequence. Am J Hum Genet. 2015 Jul 2;97(1):54-66. doi: 10.1016/j.ajhg.2015.05.005. Epub 2015 Jun 11. PMID: 26073780; PMCID: PMC4572470.

(https://www.cell.com/ajhg/fulltext/S0002-9297(15)00191-3)


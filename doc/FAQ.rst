Quiver FAQ
==========

What are EviCons? GenomicConsensus? Quiver? Plurality?  
------------------------------------------------------------
**GenomicConsensus** is the current PacBio consensus and variant
calling suite.  It contains a main driver program, ``variantCaller``,
which provides two consensus/variant calling algorithms: **Arrow** and
**Quiver**.  These algorithms can be run by calling
``variantCaller.py --algorithm=[arrow|quiver|plurality]`` or by going
through the convenience wrapper scripts ``quiver`` and ``arrow``.

**EviCons** was the previous generation PacBio variant caller (removed
 in software release v1.3.1).

Separate packages called **ConsensusCore** and **ConsensusCore2** are
C++ libraries where all the computation behind Quiver and Arrow are
done, respectively.  This is transparent to the user after
installation.


What is Plurality?
------------------
**Plurality** is a very simple variant calling algorithm: it stacks up the
aligned reads (alignment as produced by BLASR, or alternate mapping
tool), and for each column under a reference base, calls the most
abundant (i.e., the plurality) read base (or bases, or deletion) as
the consensus at that reference position.


Why is Plurality a weak algorithm?
----------------------------------
Plurality does not perform any local realignment.  This means it is
heavily biased by the alignment produced by the mapper (BLASR,
typically).  It also means that it is insensitive at detecting indels.
Consider this example::

    Reference    AAAA
                 ----
      Aligned    A-AA
        reads    AA-A
                 -AAA
                 ----
    Plurality    AAAA
    consensus

Note here that every read has a deletion and the correct consensus
call would be "AAA", but due to the mapper's freedom in gap-placement
at the single-read level, the plurality sequence is "AAAA"---so the
deletion is missed.  Local realignment, which plurality does not do,
but which could be considered as implicit in the Quiver algorithm,
essentially pushes the gaps here to the same column, thus identifying
the deletion.  While plurality could be adjusted to use a simple "gap
normalizing" realignment, in practice noncognate extras (spurious
non-homopolymer base calls) in the midst of homopolymer runs pose
challenges.

What is Quiver?
---------------
**Quiver** is a more sophisticated algorithm that finds the maximum
quasi-likelihood template sequence given PacBio reads of the template. 
PacBio reads are modeled using a conditional random field approach that
scores the quasi-likelihood of a read given a template sequence.  In
addition to the base sequence of each read, Quiver uses several
additional *QV* covariates that the basecaller provides.  Using these
covariates provides additional information about each read, allowing
more accurate consensus calls.

Quiver does not use the alignment provided by the mapper (BLASR,
typically), except for determining how to group reads together at a
macro level.  It implicitly performs its own realignment, so it is
highly sensitive to all variant types, including indels---for example,
it resolves the example above with ease.

The name **Quiver** reflects a consensus-calling algorithm that is
`QV-aware`.

We use the lowercase "quiver" to denote the quiver *tool* in GenomicConsensus,
which applies the Quiver algorithm to mapped reads to derive sequence 
consensus and variants.

Quiver is described in detail in the supplementary material to the
`HGAP paper`_.


What is Arrow?
--------------
Arrow is a newer model intended to supercede Quiver in the near future. 
The key differences from Quiver are that it uses an HMM model instead 
of a CRF, it computes true likelihoods, and it uses a smaller set of
covariates.  We expect a whitepaper on Arrow to be available soon.

We use the lowercase "arrow" to denote the arrow *tool*, which applies
the Arrow algorithm to mapped reads to derive sequence 
consensus and variants.


What data works with Arrow/Quiver?
----------------------------------

Quiver supports all PacBio *RSII* data since the C2 chemistry release
(late 2012).

Arrow supports all data from the PacBio Sequel instrument, as well as
data from the PacBio RSII using the P6-C4 chemistry.


How do I run `quiver`/`arrow`?
------------------------------
For general instructions on installing and running, see the
HowTo_ document.



What is the output from `quiver`/`arrow`?
-----------------------------------------
There are three output files from the GenomicConsensus tools:

1. A consensus *FASTA* file containing the consensus sequence
2. A consensus *FASTQ* file containing the consensus sequence with quality annotations
3. A variants *GFF* file containing a filtered, annotated list of variants identified

It is important to note that the variants included in the output
variants GFF file are *filtered* by coverage and quality, so not all
variants that are apparent in comparing the reference to the consensus
FASTA output will correspond to variants in the output variants GFF
file.

To enable all output files, the following can be run (for example)::

    % quiver -j16 aligned_reads.cmp.h5 -r ref.fa \
     -o consensus.fa                             \
     -o consensus.fq                             \
     -o variants.gff

The extension is used to determine the output file format.


What does it mean that `quiver` consensus is *de novo*?
-------------------------------------------------------
Quiver's consensus is *de novo* in the sense that the reference and the reference
alignment are not used to inform the consensus output.  Only the reads
factor into the determination of the consensus.

The only time the reference sequence is used to make consensus calls -
when the ``--noEvidenceConsensusCall`` flag is set to ``reference`` or
``lowercasereference`` (the default)- is when there is no effective
coverage in a genomic window, so Quiver has no evidence for computing
consensus.  One can set ``--noEvidenceConsensusCall=nocall`` to
avoid using the reference even in zero coverage regions.


What is the expected `quiver` accuracy?
---------------------------------------
Quiver's expected accuracy is a function of coverage and chemistry.
The C2 chemistry (no longer available), P6-C4 and P4-C2 chemistries
provide the most accuracy.  Nominal consensus accuracy levels are as
follows:

+----------+-------------------------------+
|Coverage  |Expected consensus accuracy    |
|          +------------------+------------+
|          | C2, P4-C2, P6-C4 | P5-C3      |
+==========+==================+============+
|10x       | > Q30            | > Q30      |
+----------+------------------+------------+
|20x       | > Q40            | > Q40      |
+----------+------------------+------------+
|40x       | > Q50            | > Q45      |
+----------+------------------+------------+
|60-80x    | ~ Q60            | > Q55      |
+----------+------------------+------------+

The "Q" values referred to are Phred-scaled
quality values:

.. math::
   q = -10 \log_{10} p_{error}

for instance, Q50 corresponds to a p_error of 0.00001---an accuracy
of 99.999%.  These accuracy expectations are based on routine
validations performed on multiple bacterial genomes before each
chemistry release.


What is the expected accuracy from `arrow`
------------------------------------------
`arrow` achieves similar accuracy to `quiver`.  Numbers will be published soon.


What are the residual errors after applying `quiver`?
-----------------------------------------------------

If there are errors remaining applying Quiver, they will almost
invariably be homopolymer run-length errors (insertions or deletions).



Does `quiver`/`arrow` need to know what sequencing chemistry was used?
----------------------------------------------------------------------

At present, the Quiver model is trained per-chemistry, so it is very
important that Quiver knows the sequencing chemistries used.

If SMRT Analysis software was used to build the `cmp.h5` or BAM input file, the
`cmp.h5` will be loaded with information about the sequencing
chemistry used for each SMRT Cell, and GenomicConsensus will automatically
identify the right parameters to use.

If custom software was used to build the `cmp.h5`, or an
override of Quiver's autodetection is desired,  then the
chemistry or model must be explicity entered. For example::

  % quiver -p P4-C2 ...
  % quiver -p P4-C2.AllQVsMergingByChannelModel ...



Can a mix of chemistries be used in a cmp.h5 file for quiver/arrow?
-------------------------------------------------------------------

Yes!  GenomicConsensus tools automatically see the chemistry *per-SMRT Cell*, so it
can figure out the right parameters for each read and model them
appropriately.


What chemistries and chemistry mixes are supported?
---------------------------------------------------


For Quiver: all PacBio RS chemistries are supported.  Chemistry
mixtures of P6-C4, P4-C2, P5-C3, and C2 are supported.

For Arrow: the RS chemistry P6-C4, and all PacBio Sequel chemistries
are supported.  Mixes of these chemistries are supported.



What are the QVs that the Quiver model uses?
--------------------------------------------
Quiver uses additional QV tracks provided by the basecaller.  
These QVs may be looked at as little breadcrumbs that are left behind by
the basecaller to help identify positions where it was likely that
errors of a given type occurred.  Formally, the QVs for a given read are
vectors of the same length as the number of bases called; the QVs
used are as follows:

  - DeletionQV
  - InsertionQV
  - MergeQV
  - SubstitutionQV
  - DeletionTag

To find out if your cmp.h5 file is loaded with these QV tracks, run the command
::

    % h5ls -rv aligned_reads.cmp.h5

and look for the QV track names in the output.  If your cmp.h5 file is
lacking some of these tracks, Quiver will still run, though it will
issue a warning that its performance will be suboptimal.


Why is `quiver`/`arrow` making errors in some region?
-----------------------------------------------------
The most likely cause for *true* errors made by these tools is that the
coverage in the region was low.  If there is 5x coverage over a
1000-base region, then 10 errors in that region can be expected.

It is important to understand that the effective coverage available to
`quiver`/`arrow` is not the full coverage apparent in plots---the tools
filter out ambiguously mapped reads by default.  The
remaining coverage after filtering is called the /effective coverage/.
See the next section for discussion of `MapQV`.

If you have verified that there is high effective coverage in the region
in question, it is highly possible---given the high accuracy quiver and arrow
can achieve---that the apparent errors actually
reflect true sequence variants.  Inspect the FASTQ output file to
ensure that the region was called at high confidence; if an erroneous
sequence variant is being called at high confidence, please report a
bug to us.


What does Quiver do for genomic regions with no effective coverage?
-------------------------------------------------------------------
For regions with no effective coverage, no variants are outputted, and
the FASTQ confidence is 0.

The output in the FASTA and FASTQ consensus sequence tracks is
dependent on the setting of the ``--noEvidenceConsensusCall`` flag.
Assuming the reference in the window is "ACGT", the options are:

+---------------------------------------------+---------+
|``--noEvidenceConsensusCall=...``            |Consensus|
|                                             |output   |
+=============================================+=========+
|``nocall`` (default in 1.4)                  |NNNN     |
+---------------------------------------------+---------+
|``reference``                                |ACGT     |
+---------------------------------------------+---------+
|``lowercasereference`` (new post 1.4, and the|         |
|default)                                     |acgt     |
+---------------------------------------------+---------+




What is `MapQV` and why is it important?
----------------------------------------
`MapQV` is a single scalar Phred-scaled QV per aligned read that
reflects the mapper's degree of certainty that the read aligned to
*this* part of the reference and not some other.  Unambigously mapped
reads will have a high `MapQV` (typically 255), while a read that was
equally likely to have come from two parts of the reference would have
a `MapQV` of 3.

`MapQV` is pretty important when you want highly accurate variant
calls.  Quiver and Plurality both filter out aligned reads with a
MapQV below 20 (by default), so as not to call a variant using data of
uncertain genomic origin.

This can be problematic if using quiver/arrow to get a consensus
sequence.  If the genome of interest contains long (relative to the library
insert size) highly-similar repeats, the effective coverage (after
`MapQV` filtering) may be reduced in the repeat regions---this is termed
these `MapQV` dropouts.  If the coverage is sufficiently reduced in
these regions, quiver/arrow will not call consensus in these regions---see
`What do quiver/arrow do for genomic regions with no effective coverage?`_.

If you want to use ambiguously mapped reads in computing a consensus
for a denovo assembly, the `MapQV` filter can be turned off entirely.
In this case, the consensus for each instance of a genomic repeat will
be calculated using reads that may actually be from other instances of
the repeat, so the exact trustworthiness of the consensus in that
region may be suspect.  The next section describes how to disable the
`MapQV` filter.


How can the `MapQV` filter be turned off and when should it be?
--------------------------------------------------------------
The `MapQV` filter can be disabled using the flag
``--mapQvThreshold=0`` (shorthand: ``-m=0``).  If running a
quiver/arrow job via SMRT Portal, this can be done by unchecking the "Use
only unambiguously mapped reads" option. Consider this in
de novo assembly projects, but it is not recommended for variant
calling applications.


How can variant calls made by quiver/arrow be inspected or validated?
---------------------------------------------------------------------
When in doubt, it is easiest to inspect the region in a tool like
SMRT View, which enables you to view the reads aligned to the region.
Deletions and substitutions should be fairly easy to spot; to view
insertions, right-click on the reference base and select "View
Insertions Before...".


What are the filtering parameters that quiver/arrow use?
--------------------------------------------------------

The available options limit read coverage, filters reads by `MapQV`, and filters
variants by quality and coverage.

- The overall read coverage used to call consensus in every window is
  100x by default, but can be changed using ``-X=value``.
- The `MapQV` filter, by default, removes reads with MapQV < 20.  This
  is configured using ``--mapQvThreshold=value`` / ``-m=value``
- Variants are only called if the read coverage of the site exceeds
  5x, by default---this is configurable using ``-x=value``.
  Further, they will not be called if the confidence (Phred-scaled)
  does not exceed 40---configurable using ``-q=value``.


What happens when the sample is a mixture, or diploid?
-----------------------------------------------------
At present, quiver/arrow assume a haploid sample, and the behavior of
on sample mixtures or diploid/polyploid samples is
*undefined*.  The program will not crash, but the output results are
not guaranteed to accord with any one of the haplotypes in the sample,
as opposed to a potential patchwork.  


Why would I want to *iterate* the mapping+(quiver/arrow) process?
-----------------------------------------------------------------
Some customers using quiver for polishing highly repetitive genomes
have found that if they take the consensus FASTA output of quiver, use
it as a new reference, and then perform mapping and Quiver again to
get a new consensus, they get improved results from the second round
of quiver.

This can be explained by noting that the output of the first round of
quiver is more accurate than the initial draft consensus output by the
assembler, so the second round's mapping to the quiver consensus can
be more sensitive in mapping reads from repetitive regions.  This can
then result in improved consensus in those repetitive regions, because
the reads have been assigned more correctly to their true genomic
loci.  However there is also a possibility that the potential shifting
of reads around from one rounds' mapping to the next might alter
borderline (low confidence) consensus calls even away from repetitive
regions.

We recommend the (mapping+quiver) iteration for customers polishing
repetitive genomes, and it could also prove useful for resequencing
applications.  However we caution that this is very much an
*exploratory* procedure and we make no guarantees about its
performance.  In particular, borderline consensus calls can change
when the procedure is iterated, and the procedure is *not* guaranteed
to be convergent.


Is iterating the (mapping+quiver/arrow) process a convergent procedure?
-----------------------------------------------------------------------
We have seen many examples where (mapping+quiver), repeated many
times, is evidently *not* a convergent procedure.  For example, a
variant call may be present in iteration n, absent in n+1, and then
present again in n+2.  It is possible for subtle changes in mapping to
change the set of reads examined upon inspecting a genomic window, and
therefore result in a different consensus sequence there.  We expect
this to be the case primarily for "borderline" (low confidence) base
calls.



.. _HowTo: ./HowTo.rst
.. _`HGAP paper`: http://www.nature.com/nmeth/journal/v10/n6/full/nmeth.2474.html

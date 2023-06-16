# nanopore_methylation

(Pseudo)code for ONT de novo methylation analysis

## Overview

The main tool here is [Tombo](https://github.com/nanoporetech/tombo), which is not maintained by Nanopore anymore.

This repo outlines some basic code and tricks to make it work.

## Dependencies


Conda YAML file:

```
name: env_tombo
channels:
- bioconda
- conda-forge
- r
- defaults
dependencies:
- ont-tombo
- ont-fast5-api
- ont_vbz_hdf_plugin
- numpy<1.20
- meme
```

See this [known tombo bug](https://github.com/nanoporetech/tombo/issues/394) for a fix that is not implemented in the bioconda release.

This needs to be fixed before `tombo preprocess annotate_raw_with_fastqs` can work.


### Basic workflow

1. The basic workflow starts with fast5 files (R9.4.1) which do or do not include basecalls. In my test case, I downloaded data as such:

```
# Download data from ENA
wget -nc ftp://ftp.sra.ebi.ac.uk/vol1/run/ERR895/ERR8958607/Ecoli_r9_sup.fastq.gz
wget -nc ftp://ftp.sra.ebi.ac.uk/vol1/run/ERR895/ERR8958665/Ecoli_r9.fastq.gz
wget -nc ftp://ftp.sra.ebi.ac.uk/vol1/run/ERR912/ERR9127551/ecoli_r9.tar.gz
wget -nc ftp://ftp.sra.ebi.ac.uk/vol1/run/ERR928/ERR9285206/Ecoli_MiSeq_r2.fastq.gz
wget -nc ftp://ftp.sra.ebi.ac.uk/vol1/run/ERR928/ERR9285206/Ecoli_MiSeq_r1.fastq.gz

# Unpack fast5 data
tar zxvf ecoli_r9.tar.gz

# Copy because fast5s will be edited
cp -r r9 fast5s 
```

2. I assembled the MiSeq data and sup ONT data using Unicycler to get a reference assembly.

```
unicycler -l Ecoli_r9_sup.fastq.gz -1 Ecoli_MiSeq_r1.fastq.gz -2 Ecoli_MiSeq_r2.fastq.gz -o unicycler_out -t 16
```

3. If applicable, convert multi-fast5 to single fast5s.

```
multi_to_single_fast5 -i fast5s -s fast5_single
```

4. Add fastq to the fast5 files. This MIGHT require checking read names and running `sed`. This will fail if you did not gunzip the ONT reads.

```
# Only accepts unzipped reads
gunzip Ecoli_r9_sup.fastq.gz
# Prepend "read_" to every read in ONT fastq (should match the fast5)
sed -i 's/^@/@read_/g' Ecoli_r9_sup.fastq
# Add/overwrite ONT basecalls to fast5
tombo preprocess annotate_raw_with_fastqs --overwrite --fast5-basedir fast5_single/ --fastq-filenames Ecoli_r9_sup.fastq
```
The downloaded fast5 data included basecalls but for some reason I needed to do this anyway (I think).

5. Resquiggle (align squiggle to ref genome). This will fail if you installed through conda and didn't change the source code manually.

```
tombo resquiggle fast5_single unicycler_out/assembly.fasta --num-most-common-errors 5 --processes 16 --overwrite
```

6. Detect modifications. A lot of options are available here so make sure to read the docs and help message to see what fits your needs. Multiple modifications can be specified here. This will fail if you have numpy version >=1.20.

```
tombo detect_modifications alternative_model --fast5-basedirs fast5_single --statistics-file-basename test_run --alternate-bases 6mA --processes 16
```

7. Plot the results.

```
tombo plot most_significant --fast5-basedirs fast5_single --statistics-filename test_run.6mA.tombo.stats --plot-standard-model --plot-alternate-model 6mA --pdf-filename sample.most_significant_6mA_sites.pdf
```

8. Output the context of the identified methylation to a fasta file which can be used by MEME. Again, a lot of options are available here so make sure to read them.

```
tombo text_output signif_sequence_context --fast5-basedirs fast5_single --statistics-filename test_run.6mA.tombo.stats --sequences-filename meme_input.fa
```

9. Run MEME for motif recognition.

```
meme -oc mem_out -dna -mod zoops meme_input.fa
```


### Other possibilities

If for some reason you cannot add fastq data to fast5, you need to run guppy to basecall your fast5 data AND STORE THE BASECALLS IN THE FAST5.

This can apparently only be done using older versions of guppy, e.g. v6.0.1. Taken from [Tombo issues](https://github.com/nanoporetech/tombo/issues/422#issuecomment-1448571213).

```
guppy_basecaller -i ./fast5s -s ./testing-modified-v02 -c dna_r9.4.1_450bps_fast.cfg
--as_cpu_threads_per_scaler 32 --compress_fastq --recursive
--min_qscore 10 --bam_out
--align_ref ../flye_assembly/assembly.fasta --fast5_out
```

After this, convert multi-fast5 to single fast5s and continue with `tombo resquiggle`.

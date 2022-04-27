# Public Genome Assemblies
(Motivated by a *S. aureus* example)

To have a better sense of the genomes that we have, we:

- Downloaded reads from NCBI [fasterq-dump](https://rnnh.github.io/bioinfo-notebook/docs/fasterq-dump.html)
- Ran trimomatic [trimmomatic](http://www.usadellab.org/cms/?page=trimmomatic)
- Performed *de novo* assembly with [SPAdes](https://github.com/ablab/spades)
- (Optional) Performed *de novo* assembly with [SKESA](https://github.com/ncbi/SKESA/releases)
- Performed quality checks with assemblies

### Download reads

Reads can be downloaded from NCBI efficiently with fasterq-dump (note this is a different tool from fastq-dump, which allows multi-threading).

fasterq-dump is a tool within SRA Tools, which can be installed as a conda environment, after which, the following can be run:

``` console
# set values for variables in generalized code
IDlist=/home/mwoodwo/prj/Berryhill/99.lists/IDlist.txt
outdir=/home/mwoodwo/prj/Berryhill/00.raw

for ID in `cat $IDlist`; do echo starting $ID ...; fasterq-dump $ID -t ${outdir}/fastq -e 10; echo $ID seems complete ...; done

# with comma-delimited ID file SAMPLE,SRR_ACC
for line in `cat $IDlist`; do sample=$(echo $line | cut -d, -f1); ID=$(echo $line | cut -d, -f2); echo sample is $sample    accession is $ID; if [ ! -d "${outdir}/${sample}" ]; then mkdir ${outdir}/${sample}; echo ''; echo "====================================="; echo "created ${outdir}/${sample}"; fi; echo starting $ID ...; fasterq-dump -O ${outdir}/${sample} -e 10 -p $ID; echo $ID seems complete ...; echo ''; done

```

This should produce a a directory in your specified outdir for each sample, with sub-directories for each library of reads.

We can then concatenate the 1/2 reads for each library into one sample read file as follows:

``` console
# loop over IDlist and concatenate _1.fastq and _2.fastq files for each library

for line in `cat $IDlist`; do sample=$(echo $line | cut -d, -f1); ID=$(echo $line | cut -d, -f2); echo sample is $sample    accession is $ID; cat ${sample}/${ID}_1.fastq >> ${sample}/${sample}_1.fastq; cat ${sample}/${ID}_2.fastq >> ${sample}/${sample}_2.fastq; done
```

### Trim reads

Reads can be trimmed of adaptors, minimum quality, and other parameters with trimmomatic.

- if trimmomatic was installed with conda, you may need to specify the path of illumina adaptor sequences, which are also on github:

```console
# download trimmomatic repository
git clone https://github.com/timflutre/trimmomatic

# adapters are then found in trimmomatic/adapters
adapter_path=

# add this directory to your path so they are easy to locat (optional)
export PATH="$PATH:${adapter_path}"
```

- trimmomatic can then be run:

``` console
# set values for variables in generalized code
IDlist=/home/mwoodwo/prj/Berryhill/99.lists/IDlist.txt
indir=/home/mwoodwo/prj/Berryhill/00.raw
outdir=/home/mwoodwo/prj/Berryhill/01.trimmomatic

# with comma-delimited ID file SAMPLE,SRR_ACC
for line in `cat $IDlist`; do ID=$(echo $line | cut -d, -f1); echo starting $ID ...; R1=${indir}/${ID}/${ID}_1.fastq; R2=${indir}/${ID}/${ID}_2.fastq; trimmomatic PE -threads 10 -phred33 -summary ${outdir}/summary_files/${ID}_summary.txt $R1 $R2 ${outdir}/${ID}_P1.fastq ${outdir}/${ID}_U1.fastq ${outdir}/${ID}_P2.fastq ${outdir}/${ID}_U2.fastq ILLUMINACLIP:TruSeq3-PE.fa:2:30:10:2:keepBothReads LEADING:3 TRAILING:3 MINLEN:36; echo $ID seems complete ...; echo ''; done
```

- unpaired reads can then be concatenated:

``` console
for line in `cat $IDlist`; do ID=$(echo $line | cut -d, -f1); echo starting $ID ...; cat ${outdir}/${ID}_U1.fastq ${outdir}/${ID}_U2.fastq > ${outdir}/${ID}_U.fastq; echo $ID appears complete ...; echo ''; done
```

### Assemble reads

SPAdes is one of the most commonly used de novo assemblers for isolate genomes and metagenomes.

- define variables:
``` console
IDlist=/home/mwoodwo/prj/Berryhill/99.lists/IDlist.txt    # update to just include samples once
indir=/home/mwoodwo/prj/Berryhill/01.trimmomatic
outdir=/home/mwoodwo/prj/Berryhill/02.spades
args='-t 10 -m 32' # newer versions of SPAdes don't allow --isolate flag
#args='--isolate -t 10 -m 32'
```

- run SPAdes:
``` console
for ID in `cat $IDlist`; do echo starting $ID ...; spades.py $args -1 ${indir}/${ID}_P1.fastq -2 ${indir}/${ID}_P2.fastq -s ${indir}/${ID}_U.fastq -o ${outdir}/${ID}; echo $ID complete ...; echo ''; done
```

### Check Assembly

Quast is a tool that can check assembly quality.

- define variables:
``` console
indir=/home/mwoodwo/prj/Berryhill/02.spades
outdir=/home/mwoodwo/prj/Berryhill/03.quast
IDlist=/home/mwoodwo/prj/Berryhill/99.lists/IDlist.txt
```

- run quast:
``` console
# loop for quast
for ID in `cat $IDlist`; do echo starting $ID ...; quast.py -o ${outdir}/${ID} -t 10 --glimmer ${indir}/${ID}/scaffolds.fasta ; echo $ID complete ...; echo ''; done

# run quast for all at once
quast.py -o ${outdir}/${ID} -t 10 --glimmer ${indir}/NRS225/scaffolds.fasta ${indir}/NRS111/scaffolds.fasta ${indir}/NRS242/scaffolds.fasta ${indir}/NRS112/scaffolds.fasta
```

### Test SKESA - *OPTIONAL*

There is no need to do this step routinely. Skesa claims to make higher quality assemblies compared to SPAdes but I haven't seen this to really be the case all the time. Here are the steps to construct assemblies with Skesa if you want to see if [SKESA](https://github.com/ncbi/SKESA) will produce higher quality assemblies (by N50 statistic).

- define variables:
``` console
indir=/home/mwoodwo/prj/Berryhill/01.trimmomatic
outdir=/home/mwoodwo/prj/Berryhill/04.skesa
IDlist=/home/mwoodwo/prj/Berryhill/99.lists/IDlist.txt
```

- run SKESA:
``` console
skesa --cores 10 --reads ${indir}/${ID}_P1.fastq,${indir}/${ID}_P2.fastq --contigs_out ${outdir}/${ID}_contigs.fasta
```

# Annotating variants with Hail

__14/09/2017__

![dashboard](/assets/fra.jpg)

Genomic annotating is the process of adding metadata to variants. This could be
 adding the type of variant or adding a score that indicates some pathogenicity.
 In practice a lot of these annotations are pre-computed and available on
 databases around the web. Collecting these annotation databases and writing
 scripts to add them to VCFs or other datasources can be a pain. Recently Hail
 added support for annotating variants to existing hail datasets and
 it's a breeze to use.

This feature is still in development, and as such you'll only be able to
add annotations while running on a GCP cluster.

For this demo I'm going to create a demo dataset and annotate it with all
available annotations.

## GCP account setup

You'll need a paid GCP account, the [gcloud CLI](https://cloud.google.com/sdk/docs/#install_the_latest_cloud_tools_version_cloudsdk_current_version) and be able to create [dataproc clusters](https://cloud.google.com/dataproc/).

## Starting your cluster

We'll be using a wrapper around gcloud called
[cloudtools](https://github.com/Nealelab/cloudtools). It's a great little tool
that simplifies a lot of process of managing clusters for Hail. You can install
it with pip.

```bash
pip install cloudtools
```

You can see all the defaults with:

```bash
cluster start --help
...
```

We'll start a cluster with 2 pre-emptiple workers and vep:

```bash
cluster start fred --num-preemptible-workers 2 --vep
```

With the defaults, this will produce a cluster with a total of
5 n1-highmem-8 compute instances which equates to 40CPUs and 260Gb RAM.
Approximate cost is around $2 an hour.

The cluster creation process can take a couple minutes. Once the cluster is
created, we'll connect to the jupyter notebook sever that is running on it.

```bash
cluster connect fred notebook
```

This will open a chrome browser and browse to the notebook server URL. You'll need
to navigate to a directory where you can create a notebook, by default you can
use the Cloud Storage staging bucket which will look like `dataproc-<hash>-<region>`.
Create a new notebook making sure you select the Hail kernel.

Import hail and create a test data set with 100 variants:

```python
from hail import *
hc = HailContext()

vds = hc.balding_nichols_model(3,100,100)
```

Annotate the dataset:

```python
%%time
vds = vds.annotate_variants_db([
      'va.vep',
      'va.cadd',
      'va.gene.names.ensembl_gene',
      'va.gene.names.gene_full_name',
      'va.eigen',
      'va.dann',
      'va.discovEHR',
      'va.isLCR',
      'va.gnomAD.genomes.AF_raw',
      'va.onekg.AF',
    ])
```

The descriptions of each of the annotations can be found in the [docs](https://hail.is/docs/stable/annotationdb.html).
On my cluster the annotation process took about 4min. Once the variants are annotated we can
manipulate the data further with hail or we can just export to `.tsv`.

```python
kt = vds.variants_table()
kt.export("variants.tsv")
```

The output data will look like:

```bash
head -n 2 variants.tsv
v	va
1:110:A:C	{"ancestralAF":0.6226171655765524,"AF":[0.8843877438664817,0.742222630284917,0.7314925025971879],"vep":{"assembly_name":"GRCh37","allele_string":"A/C","ancestral":null,"colocated_variants":null,"context":null,"end":110,"id":"1_110_A/C","input":"1\t110\t.\tA\tC\t.\t.\tGT","intergenic_consequences":[{"allele_num":1,"consequence_terms":["intergenic_variant"],"impact":"MODIFIER","minimised":1,"variant_allele":"C"}],"most_severe_consequence":"regulatory_region_variant","motif_feature_consequences":null,"regulatory_feature_consequences":[{"allele_num":1,"biotype":"CTCF_binding_site","consequence_terms":["regulatory_region_variant"],"impact":"MODIFIER","minimised":1,"regulatory_feature_id":"ENSR00001576074","variant_allele":"C"}],"seq_region_name":"1","start":110,"strand":1,"transcript_consequences":null,"variant_class":"SNV"},"gene":{"most_severe_consequence":"regulatory_region_variant","transcript":null,"names":{"ensembl_gene":null,"gene_full_name":null}},"discovEHR":{"AF":null},"gnomAD":{"genomes":{"AF_raw":null}},"onekg":{"AF":null},"eigen":{"raw":null,"phred":null,"PC_raw":null,"PC_phred":null},"cadd":{"RawScore":null,"PHRED":null},"isLCR":true,"dann":{"score":null}}
```

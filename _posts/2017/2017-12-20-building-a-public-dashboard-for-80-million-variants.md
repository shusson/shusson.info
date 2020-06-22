# Building a public dashboard for 80 million variants

__20/12/2017__

![billions](https://imgs.xkcd.com/comics/million_billion_trillion.png)

One area of population genomics which is quickly evolving is visualisations of large datasets. My
hope is that with good visualisations researchers will be able to find
new features and relationships in the data.

Our particular dataset is the
[Medical Genome Reference Bank (MGRB)](https://sgc.garvan.org.au/initiatives/mgrb).
At this point in time it contains approximately 80 million variants and
3000 samples. We'll be focusing on just the variants because adding sample information
adds orders of magnitude more data.

The completed dashboard is available at the [Sydney Genomics Collaborative](https://sgc.garvan.org.au/explore)(requires a google account or registration).

*update 29/12/2018*: A nice article on the challenge of scaling with genomic data was written recently, [The Desperate Quest for Genomic Compression Algorithms](https://spectrum.ieee.org/computing/software/the-desperate-quest-for-genomic-compression-algorithms).

## Backend

The first challenge we faced was choosing a database to enable the kind of visualisations
we were envisioning. The most important feature we were looking for was millisecond
 response times at the scale of 10E8 variants. We experimented with various
 databases but ended up settling on [MapD](https://www.mapd.com/).

Advantages:

- Scale to 10E8 (perhaps even 10E9) variants on a single [p2.xlarge](https://aws.amazon.com/ec2/instance-types/).
- Fast (~100ms responses for complex queries for a 10gb db with 10E8 rows on a single p2.xlarge).
- Apache 2.0 open source license.
- Includes cross-dimensional charting libraries.
- SQL interface.

Disadvantages:

- Memory constraints especially when using GPUs.
- Some limited SQL functionality.
- Requires a commercial license to horizontally scale.
- Poor at concurrent requests.

You can see some of the [tests we conducted on github](https://github.com/shusson/mapd-load-testing).

To alleviate our concerns around concurrent requests, we implemented a cache
layer. The cache layer works quite well because even though we have lots of
dimensions in the data, most of the genomics queries we see at the moment revolve around common
features like genes.

## Data processing

The data arrives in the form of a [VCF](https://en.wikipedia.org/wiki/Variant_Call_Format).
From the VCF down, all of our data processing was done using [Hail](https://github.com/hail-is/hail), which is a great open
source tool for manipulating population scale genomic data. One of the
challenges we faced was [adding annotations to the data](https://shusson.info/post/annotating-variants-with-hail).

## Deployment

The backend needs a place to live and ideally a cheap place to live. We did
a lot of testing on AWS and GCP but ended up settling on AWS because they
offer GPU spot instances which GCP doesn't support at the moment. We
created a [load balanced HTTPS service](https://shusson.info/post/creating-load-balanced-https-services-with-aws)
with autoscaling spot instances.

## Front End

The front end was built using MapD charting libraries which are a fork of
dc.js.

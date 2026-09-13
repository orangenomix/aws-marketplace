<img src="../orange_header.png" width="30%" align="left">
<br clear="left">
<hr style="border: none; border-top: 3px solid orange;">
</br>
</br>

# VEP Annotation Pipeline — User Guide

This covers launching the pipeline via CloudFormation and then submitting
VCFs to it. It doesn't cover building/publishing the container image —
see `docker/README.md` for that.

## Deployment

When you launch the stack (CloudFormation console "Create stack", or a
Marketplace pre-filled launch link), you'll be asked for:

- **BucketName** — an S3 bucket you already own, used for both input and
  output data. Just the bucket name (e.g. `my-genomics-data`), not an
  `s3://` URI or ARN. It must already exist — the stack doesn't create one.
- **VpcId** — an existing VPC in your account to run the compute in.
- **SubnetIds** — one or more subnets within that VPC for the compute
  environment. They need internet access (public subnets, or private
  subnets with a NAT Gateway) so the container image can be pulled and S3
  can be reached.
- **FilesPerJob** — maximum number of VCF files grouped into a single Batch
  job's batch (default `20`). Unlike the sibling DeepVariant pipeline, this
  pipeline isn't one job per file: every job processes its whole batch
  chromosome-major, sharing one VEP cache download and one VEP invocation
  per chromosome across every file in the batch, so a bigger batch means
  more of that shared cost gets amortized. You generally don't need to
  change this — the default is a reasonable starting point.

## Architecture

```
S3 → EventBridge → Lambda → AWS Batch → container (VEP) → S3
```

You drop VCF files into an S3 folder. That automatically groups them into
batches of up to `FilesPerJob` files and submits one Batch job per batch —
no manual job submission needed. Each job works through its batch
**chromosome-major**: for chromosome *i*, it pulls only that chromosome's
slice of the VEP cache, annotates chromosome *i*'s records across every
file in its batch in a single VEP invocation, then moves on to *i+1*
(prefetched in the background while *i* runs). This is what lets a modest
instance handle a batch without ever holding the whole genome's cache
(~26GB compressed) on disk at once.

Each input VCF is still its own independent sample with its own results —
sharing a Batch job with other files only changes how the compute is
scheduled, not the output. If you drop 20 VCFs in one folder with the
default `FilesPerJob=20`, they run as one job and you get 20 separate
annotated outputs; drop 21 and the 21st spills into a second job.

## Running a job, step-by-step

1. **Ask your admin for the S3 bucket name.**

2. **Pick a folder name** for this run, e.g. `run2026-08-26/`.

3. **Upload your VCFs.** One `.vcf` or `.vcf.gz` file per sample, directly
   under that folder:
   ```bash
   aws s3 cp sample_a.vcf.gz s3://<bucket>/run2026-08-26/
   aws s3 cp sample_b.vcf.gz s3://<bucket>/run2026-08-26/
   ```
   These should be your own called, filtered variants (e.g. PASS-only
   output from DeepVariant or another caller) — not a raw gVCF-style
   candidate-site dump. VEP annotates whatever sites you give it, so
   including non-variant candidate records (DeepVariant's `RefCall`/
   `NoCall` sites, for example) just spends compute annotating sites with
   no biological meaning.

   This same folder also needs a
   [`job_settings.yaml`](docker/job_settings.example.yaml) file (next step).

4. **Upload a `job_settings.yaml`** to the same folder. It tells the
   pipeline which VEP cache to use and any VEP options. A template with
   every option explained is at `docker/job_settings.example.yaml` — at
   minimum you need:
   ```yaml
   cache:
     species: homo_sapiens
     assembly: GRCh38
     cache_version: 116
   ```
   ```bash
   aws s3 cp job_settings.yaml s3://$BUCKET/run2026-08-26/
   ```

5. **Trigger the run** by creating an empty `_READY` file in the folder:
   ```bash
   aws s3api put-object --bucket <bucket> --key run2026-08-26/_READY --body /dev/null
   ```

6. **Check results.** Output lands in a *sibling* folder to your input
   folder — named `<input-folder>_output`, not nested inside it — with one
   subfolder per input file, named after its VCF, regardless of how many
   files shared its Batch job:
   ```
   run2026-08-26/                    <- input
     sample_a.vcf.gz
     sample_b.vcf.gz
     job_settings.yaml
     _READY

   run2026-08-26_output/             <- output (sibling, not nested inside input)
     sample_a/
       sample_a.annotated.vcf.gz
       sample_a.annotated.vcf.gz.tbi
       _SUCCESS
     sample_b/
       sample_b.annotated.vcf.gz
       sample_b.annotated.vcf.gz.tbi
       _SUCCESS
   ```
   ```bash
   aws s3 ls s3://<bucket>/run2026-08-26_output/
   #   PRE sample_a/
   #   PRE sample_b/
   ```
   Inside each: the annotated, indexed VCF (with a `CSQ` field added to
   `INFO` for every record — see the VEP documentation for the consequence
   annotation format), and either a `_SUCCESS` marker or a `_FAILURE`
   marker (containing the error) once that file's annotation finishes.
   Since several files can share one Batch job, a shared per-chromosome VEP
   failure is mirrored into *every* one of that batch's files' output
   subfolders, not just one — so checking a single file's `_SUCCESS`/
   `_FAILURE` marker always tells the whole story for that file.

## Example

```bash
BUCKET=my-genomics-bucket

aws s3 cp sample_a.vcf.gz s3://$BUCKET/run2026-08-26/
aws s3 cp sample_b.vcf.gz s3://$BUCKET/run2026-08-26/
aws s3 cp job_settings.yaml s3://$BUCKET/run2026-08-26/

aws s3api put-object --bucket $BUCKET --key run2026-08-26/_READY --body /dev/null

# wait a while, then:
aws s3 ls s3://$BUCKET/run2026-08-26_output/sample_a/
aws s3 ls s3://$BUCKET/run2026-08-26_output/sample_b/
```

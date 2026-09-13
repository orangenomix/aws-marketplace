<img src="orange_header.png" width="30%" align="left">
<br clear="left">
<hr style="border: none; border-top: 3px solid orange;">
</br>
</br>

# DeepVariant Pipeline — User Guide

This covers launching the pipeline via CloudFormation and then submitting
samples to it. It doesn't cover building/publishing the container image —
see `README.md` for that.

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

## Architecture

```
S3 → EventBridge → Lambda → AWS Batch → container (DeepVariant) → S3
```

You drop reads files into an S3 folder. That automatically triggers one
DeepVariant run per file, and the results are written back to S3 — no
manual job submission needed.

Each reads file is treated as one independent sample. If you drop 5 BAM
files in one folder, you get 5 separate runs, each with its own results.

## Running a job, step-by-step.

1. **Ask your admin for the S3 bucket name.**

2. **Pick a folder name** for this run, e.g. `run2026-08-26/`.

3. **Upload your reads.** One BAM or CRAM file per sample, directly under
   that folder:
   ```bash
   aws s3 cp patient_001.bam s3://<bucket>/run2026-08-26/
   aws s3 cp patient_002.bam s3://<bucket>/run2026-08-26/
   ```
   If you have multiple lanes/flow cells for the *same* sample, merge them
   into one file first (`samtools merge`) — each file in the folder is
   treated as a different sample.

   Uploading the index (`.bai` for a BAM, `.crai` for a CRAM) alongside its
   reads file is optional but recommended — it's picked up automatically if
   named either `patient_001.bam.bai` or `patient_001.bai` (same idea for
   `.cram`/`.crai`):
   ```bash
   aws s3 cp patient_001.bam.bai s3://<bucket>/run2026-08-26/
   ```
   If you skip this, the pipeline generates the index itself before running
   DeepVariant — just slower to start, since that adds a step.

   This same folder also needs a
   [`job_settings.yaml`](job_settings.example.yaml) file (next step).

4. **Upload a `job_settings.yaml`** to the same folder. It tells the
   pipeline where the reference genome is and any DeepVariant options.
   A template with every option explained is at
   `docker/job_settings.example.yaml` — at minimum you need:
   ```yaml
   reference:
     fasta: s3://<bucket>/refs/GRCh38/GRCh38_no_alt_analysis_set.fasta
   deepvariant:
     model_type: WGS   # or WES, PACBIO, ONT_R104, ...
   ```
   ```bash
   aws s3 cp job_settings.yaml s3://<bucket>/run2026-08-26/
   ```

5. **Trigger the run** by creating an empty `_READY` file in the folder:
   ```bash
   aws s3api put-object --bucket <bucket> --key run2026-08-26/_READY --body /dev/null
   ```

6. **Check results.** Output lands in a *sibling* folder to your input
   folder — named `<input-folder>_output`, not nested inside it — with one
   subfolder per sample, named after its reads file:
   ```
   run2026-08-26/                    <- input
     patient_001.bam
     patient_002.bam
     job_settings.yaml
     _READY

   run2026-08-26_output/             <- output (sibling, not nested inside input)
     patient_001/
       output.vcf.gz
       _SUCCESS
     patient_002/
       output.vcf.gz
       _SUCCESS
   ```
   ```bash
   aws s3 ls s3://<bucket>/run2026-08-26_output/
   #   PRE patient_001/
   #   PRE patient_002/
   ```
   Inside each: `output.vcf.gz`, a stats report, and either a `_SUCCESS`
   marker or a `_FAILURE` marker (containing the error) once that sample's
   run finishes.

## Example

```bash
BUCKET=my-genomics-bucket

aws s3 cp patient_001.bam s3://$BUCKET/run2026-08-26/
aws s3 cp patient_002.bam s3://$BUCKET/run2026-08-26/
aws s3 cp job_settings.yaml s3://$BUCKET/run2026-08-26/

aws s3api put-object --bucket $BUCKET --key run2026-08-26/_READY --body /dev/null

# wait a while, then:
aws s3 ls s3://$BUCKET/run2026-08-26_output/patient_001/
aws s3 ls s3://$BUCKET/run2026-08-26_output/patient_002/
```

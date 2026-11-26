# Amazon Inspector findings and SBOM export

This Lambda requests Amazon Inspector findings reports and SBOM exports in
multiple AWS accounts, then directs the reports to a shared S3 bucket encrypted
with a customer-managed KMS key.

The account IDs and names are currently hardcoded in `lambda.py`. Replace the
placeholder values before deploying.

## AWS resources

The setup uses:

- A Lambda function in a central/delegated account.
- A same-named IAM target role in every account listed in `lambda.py`.
- Amazon Inspector enabled and scanning resources in each target account and
  the configured Region.
- A shared S3 bucket and customer-managed symmetric KMS key in that same
  Region.

The S3 bucket policy and KMS key policy must allow the Amazon Inspector service
principal to write and encrypt reports. The example policies below are for
commercial AWS partitions (`arn:aws`). Adjust the partition and service
principal for other partitions or manually enabled Regions.

## 1. Configure the Lambda execution role

Create or use an IAM role trusted by `lambda.amazonaws.com`. Attach the
standard Lambda logging permissions (for example,
`service-role/AWSLambdaBasicExecutionRole`) and allow it to assume the target
role in every configured account:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": [
        "arn:aws:iam::<MGMT_ACCOUNT_ID>:role/<TARGET_ROLE_NAME>",
        "arn:aws:iam::<DEV_ACCOUNT_ID>:role/<TARGET_ROLE_NAME>"
      ]
    }
  ]
}
```

Keep the account IDs in this policy consistent with the account map in
`lambda.py`.

## 2. Create a target role in each account

Create the same IAM role name in every target account. Set its trust policy to
allow the Lambda execution role to assume it. Replace the principal with the
execution role ARN:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<LAMBDA_ACCOUNT_ID>:role/<LAMBDA_EXECUTION_ROLE_NAME>"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Attach this permissions policy to each target role. It covers the API actions
used by the Lambda for findings export, SBOM export, and status checks:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "inspector2:ListFindings",
        "inspector2:CreateFindingsReport",
        "inspector2:GetFindingsReportStatus",
        "inspector2:CreateSbomExport",
        "inspector2:GetSbomExport"
      ],
      "Resource": "*"
    }
  ]
}
```

The target role name must be identical across the accounts because the Lambda
constructs each role ARN as
`arn:aws:iam::<account-id>:role/<TARGET_ROLE_NAME>`.

## 3. Configure the shared S3 bucket

Create the bucket in the same Region as Inspector and the KMS key. The current
[`s3-bucket-policy.json`](./s3-bucket-policy.json) is configured for
`us-east-1`. Replace `<REPORT_BUCKET>`, `<mgmt-account-id>`, and
`<dev-account-id>`, then attach the policy to the bucket, merging its
statements with any existing bucket policy.

The policy:

- Allows the Inspector service principal to write reports for the listed
  source accounts and Region.
- Requires TLS and server-side encryption with AWS KMS for object uploads.
- Allows the dev account to list and read objects under `inspector-reports/`.

The consumer read permission is limited to the `inspector-reports/` prefix,
which matches the Lambda's default `REPORT_PREFIX`. If you choose a different
prefix, update both the dev-account read statements in `s3-bucket-policy.json`
and the resource/prefix in `target-role-policy.json`.

If the bucket is in a different account from a target account, the bucket owner
must attach this policy. The report is written under:

```text
s3://<REPORT_BUCKET>/<REPORT_PREFIX>/<findings|sboms>/<yy-mm-dd>/<account-name>/
```

Inspector chooses the report object's filename.

## 4. Configure the KMS key

Create or select a **customer-managed symmetric encryption key** in the same
Region as the bucket. The current
[`kms-key-policy.json`](./kms-key-policy.json) is configured for `us-east-1`.
Replace `<mgmt-account-id>` and `<dev-account-id>`, then apply the policy to
the key. Preserve the existing key-administrator/root permissions when
updating a key policy.

The key policy allows Inspector to encrypt exports from the listed accounts
and allows the dev account to decrypt exports. Its cross-account decrypt
permission works together with the identity policy in
[`target-role-policy.json`](./target-role-policy.json); attach that identity
policy to the dev-account role or users that need to download reports. Replace
the bucket and KMS key ARN placeholders in that file as well.

These policies are separate:

- `s3-bucket-policy.json` is attached to the shared S3 bucket.
- `kms-key-policy.json` is attached to the customer-managed KMS key.
- `target-role-policy.json` is an identity policy for report consumers; it
  grants S3 list/read and KMS decrypt/describe access.
- The Inspector API permissions from step 2 belong on each target role assumed
  by the Lambda. They are not supplied by the consumer policy.

## 5. Configure the Lambda

Set these environment variables on the Lambda:

| Variable | Required | Description |
| --- | --- | --- |
| `TARGET_ROLE_NAME` | Yes | Shared IAM target-role name in every target account. |
| `INSPECTOR_REGION` | Yes | AWS Region where Inspector is enabled and where the bucket and KMS key are located. The current bucket/key policies use `us-east-1`. |
| `REPORT_BUCKET` | Yes | Name of the shared S3 bucket. |
| `KMS_KEY_ARN` | Yes | ARN of the customer-managed symmetric KMS key. |
| `REPORT_PREFIX` | No | S3 key prefix; defaults to `inspector-reports`, matching the current consumer read policy. |
| `SBOM_FORMAT` | No | `CYCLONEDX_1_4` (default) or `SPDX_2_3`. |

Update the `accounts` dictionary in `lambda.py` with real 12-digit account IDs
and desired folder names. The placeholder IDs (`000000000000` and
`111111111111`) will not work in AWS.

## 6. Invoke and check status

An empty event requests both findings and SBOM exports for every configured
account. To request just one report type:

```json
{"report_type": "findings"}
```

```json
{"report_type": "sboms"}
```

The response contains a `report_id` and `source_account` for each requested
export. Use those values to check its status:

```json
{
  "action": "status",
  "report_id": "<REPORT_ID>",
  "account_id": "<ACCOUNT_ID>",
  "report_type": "sboms"
}
```

For findings, set `report_type` to `findings` or omit it. Findings status
checks use `GetFindingsReportStatus`; SBOM status checks use `GetSbomExport`.
Exports are asynchronous. A successful request means Inspector accepted the
request, not that the object is already available in S3.

## Notes

- The Lambda currently requests active findings in CSV format.
- Inspector permits only one findings report in progress per account. If a
  request is rejected because another findings report is running, check or wait
  for that report before retrying.
- Amazon Inspector must be enabled and have supported resources under active
  coverage in each target account and Region.
- LocalStack can help test supporting services such as STS, S3, and KMS, but
  use AWS to verify actual Inspector findings and SBOM exports.

## AWS references

- [Exporting Amazon Inspector findings reports](https://docs.aws.amazon.com/inspector/latest/user/findings-managing-exporting-reports.html)
- [Exporting SBOMs with Amazon Inspector](https://docs.aws.amazon.com/inspector/latest/user/sbom-export.html)
- [CreateFindingsReport API](https://docs.aws.amazon.com/inspector/v2/APIReference/API_CreateFindingsReport.html)
- [CreateSbomExport API](https://docs.aws.amazon.com/inspector/v2/APIReference/API_CreateSbomExport.html)
 
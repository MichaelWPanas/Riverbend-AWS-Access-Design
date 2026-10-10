# Riverbend Community Health — AWS Access Design

The IAM and storage policies behind a records portal we designed for a fictional community clinic and built on AWS. Michael Panas and Hung Hong, MSIT cloud computing project, Virginia University of Science and Technology, 2026.

**Scope, plainly:** this is a HIPAA-aware design built on synthetic data in a personal account with no BAA. It is not a HIPAA-compliant deployment. The build is a subset of the design: no load balancer, NAT gateway, RDS, Cognito, or CloudTrail was provisioned.

Case study: [link to LinkedIn Featured PDF]

## What's here

| File | What it does | Requirement |
| --- | --- | --- |
| `policies/app-tier/riverbend-s3-clinical-read.json` | App instance role: list and read `clinical/` only, decrypt with the chart key | BR-2, BR-3 |
| `policies/app-tier/riverbend-ec2-role-trust.json` | Only EC2 can assume the app role | BR-1 |
| `policies/vendor/1-billing-vendor-svc-assume-only.json` | Vendor user can do one thing: assume one role | BR-9 |
| `policies/vendor/2-billing-vendor-role-trust.json` | Assumption requires the pinned principal **and** the external ID | BR-9 |
| `policies/vendor/3-riverbend-billing-vendor-read.json` | Assumed session reads `billing/` only | BR-9 |
| `policies/vendor/4-AWSRevokeOlderSessions.json` | Kills every vendor session issued before a cutoff, staff untouched | BR-9 |
| `policies/vendor/production-hardening-encryption-context.json` | **Not deployed.** The fix for the gap below | BR-9 |
| `storage/lifecycle-archive-inactive-clinical-charts.json` | Archives `clinical/` to Glacier, with a 1-byte size filter | BR-8 |

## The vendor access chain (BR-9)

Each step was tested with a call that should succeed and a call that should fail:

1. The vendor user reads `billing/` directly. **Denied.** Data access requires the role.
2. The vendor assumes the role without the external ID. **Denied.** With it, the call succeeds.
3. The assumed session lists `billing/` (succeeds) and `clinical/` (**denied**).

## Decisions worth reading

- **KMS decrypt is granted in IAM, not the key policy.** The key's users list is empty on purpose. The default key policy delegates to IAM, so the grant lives in one place and can't drift.
- **The lifecycle rule uses a 1-byte filter.** By default, S3 won't transition objects under 128 KB. Our test charts were about 3 KB, so without the override the rule would silently do nothing. The 1-day timer is for demonstration; production uses the active window BR-8 defines.
- **The trust policy uses the in-account root plus `aws:PrincipalArn`.** The vendor's own account is fictional. It's the same confused-deputy pattern you'd use cross-account, and you can point it at a real vendor account.
- **A gap we flagged ourselves:** both read policies decrypt with the same key and no encryption-context condition, so only the S3 policies separate clinical from billing. There's no exposure today. In production, each decrypt grant gets an encryption-context condition (see the hardening file) or the prefixes get separate keys.

## Placeholders

`111122223333` is AWS's documentation account ID. `EXAMPLE-KEY-ID` and `EXAMPLE-EXTERNAL-ID` replace real values. The lifecycle rule is a JSON rendering of the console configuration. All other files are transcribed from the deployed policies.

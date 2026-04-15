---
title: "Encrypting S3 Objects with AWS KMS and Monitoring with CloudTrail"
date: 2026-04-15T16:00:00-07:00
description: "A hands-on walkthrough of encrypting S3 objects using a customer-managed KMS key and tracking every encryption operation with CloudTrail — including the gotchas that the instructions don't tell you."
tags: ["AWS", "KMS", "S3", "CloudTrail", "Security", "Encryption"]
author: "Chandra Kanth"
showTableOfContents: true
draft: false
---

## The Scenario

You're a cloud architect tasked with implementing encryption for sensitive data in S3 and setting up an audit trail for all encryption operations. Sounds straightforward, right? Create a KMS key, encrypt some objects, point CloudTrail at the bucket, done.

In practice, there are several steps that documentation and challenge instructions tend to gloss over — things that will leave you staring at "Access Denied" errors wondering what you missed. This post walks through the entire process and highlights every gotcha I hit along the way.

## What We're Building

The architecture is simple but touches three core AWS services:

1. **AWS KMS** — A customer-managed symmetric key for encrypting S3 objects
2. **Amazon S3** — A bucket with ACLs enabled, holding our encrypted objects
3. **AWS CloudTrail** — A trail logging both management events and S3 data events to the same bucket

The flow: upload a file to S3, encrypt it with your KMS key, verify you can still access it, and then confirm that CloudTrail captured all the KMS operations (Encrypt, Decrypt, GenerateDataKey) in its logs.

## Step 1: Create the KMS Key

Navigate to **KMS → Customer managed keys → Create key**.

The key configuration itself is simple:
- **Key type:** Symmetric
- **Key usage:** Encrypt and decrypt

Give it an alias — I used `whiz-kms-key` — and an optional description.

### The Part They Don't Tell You

Here's where most instructions fall short. During key creation, you'll go through two permission screens:

- **Key administrators** — who can manage (enable/disable/delete) the key
- **Key users** — who can use the key to encrypt and decrypt

**You must select your IAM user on both screens.** If you skip the key users step, you'll be able to create the key but won't be able to use it for S3 encryption later. I've seen learners complete the entire setup only to hit "Access Denied" at Step 5 because they breezed past the permissions screens during key creation.

> **Cost note:** Customer-managed KMS keys cost $1/month (prorated). If you're doing this in a personal account, schedule the key for deletion as soon as you're done — the minimum waiting period is 7 days, so don't forget or you'll eat that charge for nothing.

## Step 2: Create the S3 Bucket

Navigate to **S3 → Create bucket**.

Pick a unique name, keep the region as `us-east-1`, and here's the critical setting:

**Object Ownership → ACLs enabled → Object writer**

This is *not* the default. AWS defaults to "ACLs disabled" on new buckets, which is the recommended setting for most use cases. But this exercise specifically requires ACLs enabled with Object writer as the object owner. If you miss this, you'll need to recreate the bucket — you can't change ACL settings after creation.

I left everything else as default: Block Public Access enabled, no versioning, no default encryption override. The encryption happens at the object level in Step 5, not at the bucket level.

### A Harmless Error You'll See

After creating the bucket, you may see a red banner: *"Insufficient permissions to apply Default Encryption."* This is just the console trying to set default bucket encryption, which requires `s3:PutEncryptionConfiguration` — a permission the lab account doesn't have. Dismiss it. It's irrelevant because you're encrypting individual objects, not setting bucket-level defaults.

## Step 3: Create the CloudTrail Trail

Navigate to **CloudTrail → Trails → Create trail**.

This step has the most configuration:

**Trail attributes:**
- **Trail name:** `whiz-kms-trails`
- **Storage location:** Use an existing S3 bucket (select the one from Step 2)
- **Log file SSE-KMS encryption:** Uncheck this
- **Log file validation:** Leave enabled (default)
- **CloudWatch Logs:** Leave disabled

**Log events:**
- **Management events:** Read and Write (both checked)
- **Data events:** This is the tricky part

### Configuring Data Events

The data events UI has changed over time. You might see "Advanced event selectors" enabled by default. Here's what to do:

1. Select **S3** as the data event source
2. Change the log selector template from "Log all events" to **Custom**
3. Set the field to **resources.ARN**
4. Operator: **starts with**
5. Value: `arn:aws:s3:::your-bucket-name/` (include the trailing slash)

This tells CloudTrail to log all S3 data operations (GetObject, PutObject, etc.) specifically for objects in your bucket, rather than logging every S3 event across your entire account.

If you find the advanced selectors confusing, look for the **"Switch to basic event selectors"** button — the basic view gives you a simpler interface where you can just browse and select your bucket.

You'll also see a "Configure event aggregation" step — skip it, just click Next and create the trail.

## Step 4: Upload a Test File

Navigate to your S3 bucket and click **Upload**. Any small file works — I uploaded a simple `.txt` file with a few lines of text. The file type doesn't matter; what matters is that we have an object to encrypt.

Don't change any encryption settings during upload — we'll handle that in the next step.

## Step 5: Encrypt the Object with KMS

This is the core of the exercise, and it's also where the instructions tend to be vague. "Encrypt the object using the KMS key" — but how?

There's no "Encrypt" button. Here's the actual path:

1. Click on your uploaded file in S3
2. Go to the **Properties** tab
3. Scroll to **Server-side encryption settings**
4. Click **Edit**
5. Select **"Override bucket settings for default encryption"**
6. Choose **SSE-KMS** (Server-side encryption with AWS KMS keys)
7. Under "AWS KMS key," select **"Choose from your AWS KMS keys"**
8. Pick your key (`whiz-kms-key`)
9. Click **Save changes**

What happens behind the scenes: S3 re-encrypts the object using your KMS key. The file content doesn't change — only the encryption envelope. It's a metadata operation, not a re-upload. After this, any access to the object will require `kms:Decrypt` permission on the key.

## Step 6: Access the Encrypted Object

Now try to download or open the file from the S3 console. Click on the object and hit **Download** or **Open**.

**Important:** Use the console buttons, not the Object URL directly. The Object URL is an unauthenticated link — it will always return "Access Denied" for non-public objects regardless of encryption. This confused me initially because I clicked the URL and assumed the KMS encryption was blocking me.

If you set up key permissions correctly in Step 1 (your user as a key user), the console download will work seamlessly. The S3 console silently calls `kms:Decrypt` using your credentials.

### What If It Doesn't Work?

If you get Access Denied even through the console, check:
- **KMS → Customer managed keys → your key → Key policy tab** — verify your user is listed under "Key users"
- If you're in a personal account, you can edit the key policy directly to add `kms:Decrypt`, `kms:DescribeKey`, and `kms:GenerateDataKey` permissions for your user

> **Blog-worthy experiment:** In a personal account, deliberately remove your decrypt permission, observe the error, then fix it. Screenshots of the before/after make excellent teaching material.

## Step 7: Analyze CloudTrail Logs

CloudTrail logs take 5–15 minutes to appear in the S3 bucket. Don't panic if you check immediately and see nothing.

### Where to Find the Logs

Navigate to your S3 bucket. You'll see a new **AWSLogs/** folder that CloudTrail created automatically. Drill into:

```
AWSLogs/
  └── <account-id>/
      └── CloudTrail/
          └── us-east-1/
              └── 2026/
                  └── 04/
                      └── 15/
                          └── <account-id>_CloudTrail_us-east-1_20260415T....json.gz
```

These are compressed JSON files containing every API call CloudTrail captured. Download one, decompress it, and look for KMS-related events:

- **`Encrypt`** — when you applied KMS encryption to the object
- **`Decrypt`** — when you downloaded/accessed the encrypted object
- **`GenerateDataKey`** — when S3 requested a data key from KMS for envelope encryption

### Alternative: CloudTrail Event History

If you have `cloudtrail:LookupEvents` permission, you can also check **CloudTrail → Event history** and filter by Event source `kms.amazonaws.com`. This is faster than digging through `.json.gz` files — but not all lab environments grant this permission. In my case, Event History was blocked while the S3 logs worked fine.

## Lessons Learned

### 1. KMS Key Permissions Are Set During Creation — Not After

The key creation wizard walks you through administrator and usage permissions. If you click "Next" without selecting your user, the key exists but is essentially useless to you. This is the number one gotcha for this exercise.

### 2. "Encrypt the Object" Means Editing Object Properties

There's no standalone "Encrypt" action in S3 or KMS. You change encryption by editing the object's server-side encryption settings in its Properties tab. This isn't intuitive if you're looking for a dedicated encryption workflow.

### 3. Object URLs Always Fail for Non-Public Objects

The Object URL in S3 is an unauthenticated HTTP link. It will return Access Denied for any object that isn't explicitly public — encryption or not. Always use the console's Download/Open buttons for authenticated access.

### 4. CloudTrail Logs Have a Delay

Expect 5–15 minutes before logs appear in S3. If you're validating a challenge, build in wait time rather than assuming something is broken.

### 5. Not All Console Errors Are Problems

During this exercise I saw at least three "Access Denied" or "Insufficient permissions" banners that were completely irrelevant to the task:
- `s3:PutEncryptionConfiguration` on bucket creation (cosmetic — default encryption isn't needed)
- `iam:ListUsers` when checking IAM (expected — labs lock down IAM introspection)  
- `cloudtrail:LookupEvents` for Event History (workaround: check logs directly in S3)

Learning to distinguish real blockers from harmless console noise is a skill in itself.

## Cost Breakdown (Personal Account)

| Resource | Cost | How to Clean Up |
|----------|------|-----------------|
| KMS Key | $1/month (prorated) | Schedule deletion (7-day min wait) |
| CloudTrail (data events) | $0.10 per 100K events | Delete the trail |
| S3 Bucket | Minimal (storage only) | Empty and delete |
| **Total for this exercise** | **< $1.00** | **Clean up immediately after** |

## Final Thoughts

This exercise covers a surprisingly practical pattern: customer-managed encryption with audit logging. In production, this exact setup — KMS keys with strict key policies, S3 encryption, and CloudTrail monitoring — forms the backbone of compliance frameworks like HIPAA, SOC 2, and PCI-DSS.

The technical steps aren't hard. What's hard is knowing which permissions to set during key creation, where to find the "encrypt" option in S3, and how to interpret CloudTrail logs once they land. Hopefully this walkthrough saves you the time I spent figuring those things out.

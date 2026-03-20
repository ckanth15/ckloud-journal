---
title: "Debugging AWS Lambda: Why Your Zip File Structure Matters"
date: 2026-03-19
description: "How a nested zip file and a Python runtime mismatch caused a frustrating ImportModuleError — and how I fixed it."
tags: ["AWS", "Lambda", "RDS", "MySQL", "Python", "Troubleshooting", "Serverless"]
categories: ["AWS Projects"]
ShowToc: true
TocOpen: false
---

## What I Was Building

Query an RDS MySQL database using AWS Lambda with Python and PyMySQL. Sounds straightforward — create an RDS instance, populate it with data via MySQL Workbench, write a Lambda function to query it, done.

It wasn't that simple.

## The Error

After deploying the Lambda function and hitting Test, I got this:
```json
{
  "errorMessage": "Unable to import module 'lambda_function': No module named 'lambda_function'",
  "errorType": "Runtime.ImportModuleError",
  "requestId": "",
  "stackTrace": []
}
```

The function was deployed, the code looked correct, the handler was set to `lambda_function.lambda_handler`. So why couldn't Lambda find the module?

## Investigation

### Checking the Zip Structure

The lab provided a pre-built zip file containing `lambda_function.py` and the `pymysql` dependency. I extracted it and the files looked fine locally:
```
RDS_SQL_query/
├── lambda_function.py
├── pymysql/
├── PyMySQL-0.10.1.dist-info/
├── template.yaml
└── lambda-payloads.json
```

But then I ran `unzip -l` to inspect the actual zip paths:
```bash
unzip -l RDS_SQL_query.zip | head -10
```
```
RDS_SQL_query/lambda_function.py
RDS_SQL_query/pymysql/
RDS_SQL_query/pymysql/charset.pyc
...
```

**There it was.** Every file was nested inside an `RDS_SQL_query/` parent directory. Lambda expects `lambda_function.py` at the **root** of the zip, not inside a subdirectory.

### The Runtime Mismatch

The zip also contained `.pyc` bytecode files compiled for Python 3.8 (the `template.yaml` confirmed `Runtime: python3.8`). I had selected Python 3.13 — the latest available. While PyMySQL is pure Python, the stale `.pyc` files and version gap can cause additional import issues.

## The Fix

**1. Re-zip from inside the directory:**
```bash
cd ~/Downloads/RDS_SQL_query
zip -r ../lambda_package_fixed.zip . -x "template.yaml" "lambda-payloads.json"
```

This puts `lambda_function.py` at the zip root — exactly where Lambda expects it.

**2. Switch runtime to Python 3.10** (closest available to the original 3.8, while 3.9 wasn't offered).

**3. Upload, Deploy, Test** — success on the first try.

## Key Takeaways

1. **Always inspect zip structure before uploading to Lambda.** Run `unzip -l yourfile.zip` to verify handler files are at the root, not nested in a subdirectory.

2. **Runtime version matters for dependencies.** If a package bundles `.pyc` files for a specific Python version, don't jump multiple major versions ahead.

3. **`pymysql.connect()` needs keyword arguments.** Newer versions require `host=endpoint` rather than passing the endpoint as a positional argument.

4. **Increase Lambda timeout.** The default 3 seconds often isn't enough for initial RDS connection handshake — set it to at least 30 seconds.

## Architecture

The final working setup:

- **RDS**: MySQL on db.t3.micro, publicly accessible, security group allowing port 3306
- **Lambda**: Python 3.10, PyMySQL 0.10.1, 128 MB memory, 30-second timeout
- **Query**: SELECT and INSERT operations against a StudentDB database

---

*This was a lab assignment where the debugging experience turned out to be more valuable than the lab itself — real-world Lambda packaging issues are exactly the kind of thing that trips people up in production.*

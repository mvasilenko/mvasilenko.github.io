---
title: "How to restore S3 objects from Glacier to Standard Storage"
date: 2023-10-03T23:59:57+01:00
draft: false
---

Let's imagine you have objects archived in S3 Glacier tier and you got a request made them available. For this the restoration is required - because Glacier Archive tier is a magnetic tape, and restoration is a batch job, there will be few hours delay after restore initiation.

Commands - taken from https://motilevy.medium.com/permanently-restore-s3-objects-from-glacier-f4b88503a5e6


Create objects list, which needs to be restored

```
aws s3api list-objects-v2 \
 — bucket <bucket-name> \
 — query “Contents[?StorageClass==’GLACIER’]” \
 — output text | awk ‘{print $2}’ > glacier_files.txt
```

Initiate the restore

```
cat glacier_files.txt| while read file
do 
  echo $file && aws s3api restore-object \
   --restore-request Days=25 \
   --bucket <bucket-name> --key $file
done
```

Monitor restore status

```
cat glacier_files.txt| while read file
do
echo -n "$file: " && aws s3api head-object \
  --bucket <bucket-name> \
   --key $file | jq .Restore
done
```

A completed restore will show the request as `pending=False`

```
"ongoing-request=\"false\", expiry-date=\"Sat, 04 Mar 2023 00:00:00 GMT\""
```


Copy objects over themselves, changing the storage tier

```
cat glacier_files.txt| while read file
do
aws s3 cp s3://<bucket-name>/$file s3://<bucket-name>/$file  \
   --storage-class STANDARD \
   --force-glacier-transfer
```
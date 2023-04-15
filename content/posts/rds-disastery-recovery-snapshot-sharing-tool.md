---
title: "RDS Disaster Recovery using Snapshot sharing tool"
date: 2023-04-15T22:34:03+01:00
draft: false
---

Lets imagine the scenario, when attacker gains access to the production AWS account and can modify or remove RDS databases and snapshots, affecting business continuity. 

To mitigate such risks we should have disaster recovery solution is in place in the form of an RDS snapshot tool that runs on a daily basis in two accounts in parallel: the production account and the backup account.

Production account: RDS snapshot tool creates encrypted snapshots of the RDS databases and recrypts them with a KMS key shared from the backup account. The tool then shares the encrypted snapshots with the backup account. 

Backup account: RDS snapshot tool copies the encrypted snapshots from the production account to the backup account, rotating and keeping only a few last snapshots.

Even when attacker gains access to the production AWS account, he is unable to access the backups and the data stored within them.

![RDS snapshot tool diagram](https://i.stack.imgur.com/wWtVS.png)

Github repos implementing RDS snapshot tool for RDS PostgreSQL and RDS Aurora PostgreSQL

https://github.com/mvasilenko/disaster-recovery-rds-aurora

https://github.com/mvasilenko/dr-rds-share-snapshot


 [Stackoverflow question](https://stackoverflow.com/questions/59983969/aws-rds-disaster-recovery-using-cross-account/67905878#67905878) describing RDS share snapshot tool 
 
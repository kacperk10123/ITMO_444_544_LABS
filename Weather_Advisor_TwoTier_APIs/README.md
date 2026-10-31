# Lab 06: Weather Advisor (two-tier app on AWS)


A weather service and an AI clothing advisor, built with bash scripts on AWS. Each tier runs in its own Auto Scaling Group behind one Application Load Balancer. Weather lookups are saved in a private, encrypted RDS PostgreSQL database. Secrets live in SSM Parameter Store, and the advisor calls Gemini with a fallback if Gemini fails. I built it, tested it, broke it on purpose, and tore it all down.


## AI use
Claude walked me through any problems and assisted me with troubleshooting. I ran every command myself, captured every screenshot, and fixed the errors listed under Troubleshooting. The Gemini API key was created by me in my own Google account.


## Learning objectives
- Keep secrets out of files and user-data by storing them as SSM SecureStrings, and give each IAM role access to only its own secret.
- Run a database that is private, encrypted and single-AZ, and understand what turning backups off (retention 0) costs and saves.
- Use security groups that allow traffic only from another security group, not from an IP range.
- Build graceful degradation: a cache backed by the database, a 30-second pause when the database is down, and a fallback answer when Gemini fails.
- Measure recovery time when an instance is killed, and see why anything important must live outside the instance.
- Write idempotent create and delete scripts, then prove nothing is left running or billing.


## Troubleshooting
| Problem | Fix |
|---|---|
| SSM and RDS commands failed with AccessDenied for my lab user | I own the account, so I attached the AmazonSSMFullAccess and AmazonRDSFullAccess managed policies to `lab-user` in the console. |
| Gemini returned a 402 billing error | Created a new key from a new Google project (free tier) and stored it in SSM. |
| After the Step 7 database test, the weather answers kept saying `cache = unavailable` | I had left the database security group rule revoked. I added the rule back and the cache returned. |
| Some stack scripts did not run as pasted | I pasted the scripts again and reran them. The scripts are safe to run twice. |
| The `create_database.sh` log looked garbled | The script had run before, so the rerun showed each step as "already exists, skipping". The Step 4 table proves the final settings. |

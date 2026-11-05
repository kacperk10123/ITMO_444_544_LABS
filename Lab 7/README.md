# Lab 07: Infrastructure as Code with Terraform (Weather Advisor)

I rebuilt the lab06 weather advisor stack (two tiers, one ALB, two Auto Scaling Groups, DynamoDB, IAM roles, and a Gemini key in Parameter Store) in about 430 lines of Terraform, with no delete script. Then I changed it, broke it on purpose, and destroyed it with one command.

## AI use
Claude walked me through any troubleshooting I had in this lab step by step. It also helped me format this lab report so it's more readable. I ran every command myself, captured every screenshot, read every plan and fixed the problems listed below. I created the Gemini key in my own personal Google account.

## Learning objectives
- Explain imperative code (my bash scripts) and declarative code (Terraform), and what each is best at.
- Run the whole workflow: `init`, `fmt`, `validate`, `plan`, `apply` and `destroy`.
- Read a plan and say what will be created (`+`), updated in place (`~`), or destroyed and created again (`-/+`).
- Use variables, outputs, data sources, `for_each` and `lifecycle` to avoid repeating myself.
- See that Terraform works out the order itself from the references between resources.
- Detect and repair drift, and roll out a code change with an instance refresh.
- Explain how Terraform is like AWS Lambda and where the comparison stops.
- Destroy everything with one command and prove nothing that costs money is left.

## Troubleshooting
| Problem | Fix |
|---|---|
| `terraform plan` failed with "Attribute redefined" for `cache_seconds` | I had run the `echo 'cache_seconds = 60' >> terraform.tfvars` line several times, so the setting appeared four times. I deleted all the copies and added one back. |
| The first smoke test failed only the Gemini check (the advisor took 15.1 seconds and answered with the fallback) | The app waits 15 seconds for Gemini, so Gemini was slow once. I ran the test again and it passed 9 of 9, with a 0.7 second answer. |
| The copied `lab07/previous/` folder held old scripts from an earlier lab, which would have inflated my line counts | I deleted that folder and created a fresh `previous/` holding only my four lab06 scripts. |
| After I deleted the advisor listener, the page still answered 200 | The load balancer takes a few seconds to stop serving a deleted listener. I waited and ran it again, and the page returned no answer. |
| The lab's check `grep -c "AIza"` on the state file printed 0 and I expected to see the placeholder | My key doesn't start with `AIza`, so the 0 was misleading. A Python check that never printed the key showed the state held a 53 character value equal to my real key in SSM. |



# Lab 07 Written Answers
 
## Part A: Terraform first taste
 
The `-/+` plan was the one to be careful about, because it destroys the table and all its data and then creates a new one. DynamoDB can't change a table's key, and the plan warns me before it deletes anything.
 
## Part E: Smoke test
 
The single line in `compute.tf` that made Terraform build the load balancer before the advisor's launch template is `WEATHER_URL = "http://${aws_lb.main.dns_name}:..."` in the advisor's settings. It refers to `aws_lb.main`, and that reference is the whole ordering logic.
 
## Step 1: Count the effort
 
| | lab06 bash | lab07 Terraform |
|---|---|---|
| Lines to **create** the stack | 363 | 432 |
| Lines to **delete** the stack | 121 | 0 |
| Steps I had to order by hand | 12 | 0 |
| Helper functions I wrote | 6 | 0 |
| Retry or wait loops I wrote | 9 | 0 |
 
Terraform was not shorter to write, with 432 lines against 363 to create the stack, so the saving isn't in line count. The effort went down in the work around the resources: I wrote no 121 line delete script, no order of steps, no "does it exist already" helper functions, and no wait loops, because Terraform worked those out itself.
 
## Step 2: The order Terraform worked out
 
If I removed the `WEATHER_URL` line from `compute.tf`, the arrow from the launch template to `aws_lb.main` would disappear, because that one reference is the only thing that made Terraform build the load balancer first. The order could change, and the advisor would not work, because it wouldn't know the address of the weather service and would return a 502.
 
## Step 3: Run plan again
 
Running `terraform plan` again said `No changes` after comparing my files with the state and with the real AWS resources. Running `create_weather_stack.sh` twice only checked that a name existed and printed "already exists, skipping", and it never checked whether the existing resource still had the right settings.
 
## Step 4: Change one setting
 
With Terraform this took two commands, editing `terraform.tfvars` and running `terraform apply`, and the instance was replaced for me. In lab06 it took four: edit `.env`, run the create script, find the instance, and terminate it myself.
 
## Step 5: Drift
 
In lab06, `ensure_target_group` only checked that a group with that name existed, said "skipping", and never checked that the health check was still right. In Terraform, `terraform plan` compared my code with the real AWS settings, saw the hand-made `/nope` path, and showed a `~` to set it back to `/`. It also noticed the deleted listener and planned `1 to add`.
 
## Step 6: Least privilege
 
An attacker who takes over the advisor tier can read the one Gemini key (and use up its quota), but they cannot read or change the weather table or any saved lookups, because the advisor role is allowed to read only that one parameter.
 
## Step 7: The key and Terraform
 
The plan said `No changes` even though the stored value is different from `not-set-yet`, so `ignore_changes` works: Terraform does not compare or overwrite the value.
 
Without `ignore_changes`, the next `terraform apply` would overwrite my real key with the placeholder, and the advisor would fall back to rule-based answers.
 
In my own state file I found that Terraform had copied the real key in when it refreshed the parameter. A Python check that never printed the key showed the state held a 53 character value that was not the placeholder and was equal to my real key in SSM. The `grep -c "AIza"` printed 0 only because my key doesn't start with `AIza`. So `ignore_changes` protects the key from being overwritten, but it does not keep it out of the state, and the state file has to stay private and out of Git.
 
## Step 8: The fallback
 
With a design where users never see an error, the only evidence of a broken key is a log line that nobody reads, so a bad or expired key could go unnoticed for weeks while every answer is lower quality. I'd want an alert on how often the fallback is used.
 
## Step 9: The Lambda exercise
 
| Advisor on EC2 (this lab) | Advisor on Lambda | Effort moved to |
|---|---|---|
| Launch template and user-data | One `aws_lambda_function` resource with the code zipped | AWS runs the servers, so I no longer write a boot script. I take on packaging the code and its libraries into a zip |
| Auto Scaling Group and instance refresh | Not needed. Lambda starts copies as requests come in, and a new code version replaces the old one | AWS. The new problem is scaling limits and throttling |
| App security group rule for port 3000 | Not needed. There's no port, only a permission (`aws_lambda_permission`) that lets the load balancer call the function | IAM permissions |
| Instance profile and role | An execution role with a Lambda trust policy. The `ssm:GetParameter` policy stays the same | Still mine (IAM), with a different trust policy |
| Node.js install and systemd service | Not needed. AWS provides the runtime and restarts it for me | AWS. The new problem is that my code must match a supported runtime and handler shape |
| Health check and grace period | Not needed. A Lambda target group has health checks off by default, and there's no boot time to wait for | AWS. The new problem is timeouts and cold starts |
 
1. **Exists only because of EC2:** the launch template and user-data, the Auto Scaling Group and instance refresh, the app security group and its three rules, the AMI lookup, the instance type and key pair variables, and the target group's health check settings.
2. **Stays:** the SSM parameter and the advisor's `ssm:GetParameter` policy, the listener on port 8081, the load balancer and its security group, the weather tier and DynamoDB, and the model and key-cache settings.
3. **New:** an `aws_lambda_function`, a permission that lets the load balancer call it, a target group of type `lambda`, an execution role, and code changes so the app handles a load balancer event and returns a response instead of running an Express server.
4. **Effort:** Terraform removed the effort of ordering, waiting and deleting. Lambda would remove the effort of servers, including the launch template, the Auto Scaling Group, the OS and Node.js install, and the port rule. What I would gain is no instance to patch or replace and paying only when the function runs. The new problems would be cold starts (a slow first request after idle), time and payload limits, rewriting the app to handle a load balancer event, and testing the function by sending it a fake event instead of just calling a URL.
## Short answers
 
**How Terraform is like Lambda, and where the comparison stops.** Both let me hand over a small, self-contained piece and let the platform do the repetitive, error-prone work: ordering and retries for Terraform, servers and scaling for Lambda. Both act through a role, run on demand and then stop, and both are idempotent when written well. The comparison stops because Lambda is a runtime service that runs my application code at request time, while Terraform is a deployment tool that runs before my application exists and never handles a request. Terraform also keeps a state file that I have to protect, and the tool is free while the resources it creates cost money.
 
**What Terraform did not make easier.** It did not make the code shorter, with 432 lines against 363 to create the stack. I had to learn a new language (HCL), a workflow and a state file. Storing the real Gemini key is still a one-time command in the CLI. The state can leak secrets, and mine held my real key. Mistakes are fast, because a single `terraform destroy` removes everything it manages.
 
## Section 13.1 Review questions
 
1. **Declarative and imperative:** Bash can run any one-time or custom step in order, such as typing the real Gemini key at a hidden prompt, running `ssh` checks, or choosing what to do when a step fails, which is hard to describe in Terraform because it describes the end state, not steps. Terraform showed me a plan before changing anything, noticed drift, and destroyed everything with one command, none of which my bash scripts did.
2. **State file:** If I lost it, Terraform wouldn't know it had built anything, so it would try to create everything again and fail on names that already exist, and I'd have to find and delete the leftovers by hand (the `Project` tag helps). If two people applied from different copies, each would work from a different idea of reality and could overwrite each other's changes or create duplicates. Teams keep the state in one shared, locked place, for example an S3 bucket with locking.
3. **Secrets in state:** A secret can reach the state any time Terraform reads or writes it, for example through a resource's value, a data source, or an output, and I found my real key in the state after Terraform refreshed the parameter. A plan file is also sensitive, because it contains the same values and can be applied.
4. **Plan before apply:** Reading the plan protects me from surprises, such as something being destroyed that I didn't expect. The `-/+` symbol worries me most, because it destroys a resource and creates it again, which loses data and causes downtime, as the DynamoDB `hash_key` change showed.
5. **Version pinning:** With no pin, a new provider version could change behavior or rename arguments and break my plan or apply without me touching the code. The lock file `.terraform.lock.hcl` records the exact version, so everyone on the team gets the same one.
6. **`for_each` or `count`:** `for_each` names each copy by its key, such as `tier["weather"]`, so removing one entry removes only that copy. With `count` the copies are numbered 0, 1, 2, so removing the first one shifts the others and Terraform could destroy or recreate the wrong ones.
7. **Ownership:** When a tool and a person both edit the same resource, they undo each other: the next `apply` quietly reverts the person's change, as it did with my `/nope` health check, or the change causes an outage if nobody notices. Only one of them should own each resource.
8. **Drift:** A team can run `terraform plan` on a schedule and alert whenever it isn't empty. Hand changes should be allowed only for a few trusted people in an emergency, and then written back into the code.
9. **Blast radius:** What a change touches is decided by what references the changed value. `cache_seconds` only fed the weather tier's user-data, so only the weather launch template and Auto Scaling Group changed. I could shrink it further by splitting the stack into separate configurations and state files per tier.
10. **Least privilege:** A reference stays correct if the resource is renamed or recreated, because Terraform fills in the right ARN and builds the dependency, while a typed-in ARN can contain a typo or point to the wrong resource without any warning, like the lab06 `AccessDenied`.
11. **Terraform and Lambda:** They share the idea of handing over a small piece and letting the platform do the repetitive work, and both act through a role. The one difference that matters most is that Lambda runs my code at request time, while Terraform builds the infrastructure beforehand and never handles a request. I would use Terraform to create and maintain the stack and Lambda to run the application code.
12. **Destroy:** I would protect anything holding data that can't be recreated: databases, S3 buckets with data, DynamoDB tables with real data, encryption keys, and the bucket that holds the state. `lifecycle { prevent_destroy = true }` makes any plan that would destroy them fail with an error.
## Step 10: Destroy
 
The stack started at about 12:02 local time (18:02 UTC). I ran `terraform destroy` at 13:13 local time and it finished at 13:20. The destroy plan said `Plan: 0 to add, 0 to change, 25 to destroy.` and the result was `Destroy complete! Resources: 25 destroyed.` It took 6 minutes 43 seconds, and almost all of that was the two Auto Scaling Groups, which took 6 minutes 5 seconds because they wait for their instances to terminate. This one command replaced my 121 line delete script.
 
The stack ran for about 1 hour 19 minutes (1.3 hours). At about $0.07 an hour, my cost estimate is about 9 cents ($0.09).
 
After the destroy, the verification commands showed nothing left: `terraform state list` was empty, and the load balancers, Auto Scaling Groups, running instances, DynamoDB tables, SSM parameters and `itmo-444-544` IAM roles all came back empty.

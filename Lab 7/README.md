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

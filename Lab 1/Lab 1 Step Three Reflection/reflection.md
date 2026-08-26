One paragraph explaining why you should never commit AWS credentials or GitHub tokens to a repository, even a private one.

Never commit AWS credentials or GitHub tokens to a repository, even a private one, because once they're in your git history they're extremely hard to fully remove. 
Even if you delete the file later, the secret still lives in old commits. Private repos aren't as safe as people think too. This is because collaborators, CI/CD logs, forks, or a simple mistake can expose that history to the wrong people. 
Automated bots constantly scan GitHub for exposed keys and tokens, and they can find and abuse leaked credentials within minutes, even on repos you thought were private. 
If someone malicious gets your AWS keys, they can spin up expensive cloud resources or access sensitive data which can sometimes cost you hundreds or thousands of dollars before you notice. 
A leaked GitHub token can let someone push malicious code, delete repositories, or access other private projects tied to your account. The safe practice is to keep secrets out of your code entirely.

One paragraph comparing ignore_errors("EntityAlreadyExists") with 2>/dev/null || true .

2>/dev/null || true means “if anything goes wrong, hide it and keep going.” 
It hides every possible failure, not just the harmless one I expected. 
A typo, a missing permission or the wrong region would all be swallowed, and the script would carry on as if everything was fine. Ignore_errors("EntityAlreadyExists") only ignores one specific error, which is “this role already exists” and is safe to skip. 
If anything else goes wrong, Python raises the error and stops, so I find out right away. That makes the Python version more honest, because it only stays quiet when staying quiet is correct.

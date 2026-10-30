# Lab 06: Written Answers

### Part A
The weather service makes two upstream calls, a zip lookup that turns the zip into coordinates and then the NWS forecast, because NWS only accepts latitude and longitude, never zip codes.

### Step 1
In `{"event":"weather_success","zip":"30301","cache":"miss","ms":239}`, the request for zip 30301 succeeded, it wasn't in the database yet (a miss), and the server spent 239 ms on it.

### Step 3
The miss took 239 ms and the hit took 2 ms. A short cache matters more for a service that calls a rate-limited upstream, because repeat requests never reach the upstream and don't use up its limit. `CACHE_SECONDS` shouldn't be long for weather because forecasts change, and a long cache would serve stale data.

### Step 5
An attacker who takes over the advisor tier gets the Gemini key (and can use up its quota) and can reach the database port, because both tiers share one security group. They do not get the database password.

### Step 6
A rule that names another security group as its source allows only traffic from members of that group, which can't be faked by picking an IP address, while a CIDR range allows anyone whose address falls inside it.

### Step 7
Without the 30-second pause, every request while the database is down would wait the full 3-second timeout before giving up, which slows every answer and keeps hammering a database that is already failing. In my test, three requests produced only two `db_error` lines, because the middle one skipped the database and answered at full speed.

### Step 8
With the fallback, users never see an error, and the only evidence is a log line that nobody reads, so a broken key could go unnoticed for weeks. I'd want an alert on how often the fallback is used.

### Step 9
Recovery took 69 seconds (my other runs were 69 s and 84 s). The advisor returned a clean 502 during the outage, and afterward the new instance answered for zip 02101 from a lookup the old instance had saved. Anything you can't afford to lose should live in a managed service outside the instance, because the instance's own disk disappears when it is replaced.

## Paragraphs

### Why the secrets are in SSM, each read by one role
The Gemini key and the database password are kept in SSM Parameter Store as encrypted SecureStrings, not in `.env` or the user-data. User-data can be read by anyone who can describe the launch template, and a `.env` file can get committed to Git by accident. Each instance reads its secret at run time using its IAM role, so the secret never sits in any of my files. Each role can read only one parameter: the weather role reads the database password and the advisor role reads the Gemini key. I proved this with the IAM policy simulator, which showed `allowed` for each role's own secret and `implicitDeny` for the other. That way, if one tier is taken over, the attacker gets one secret and not both.

### Turning backups off
I set backup retention to 0, which turns off automated backups and point-in-time recovery. This saved backup storage and meant no snapshot was left behind when I deleted the database, so my cleanup check showed no snapshots or backups remaining. The cost is that the database can't be restored if it is damaged or deleted. That was fine here, because the data is public forecasts plus a history table I created while testing, and the whole database was deleted at the end. I would never do this for data I care about, like customer records. For that I would keep several days of automated backups and practice restoring them.

### What I sent to Gemini and why the free tier matters
I sent only the zip code, the city and state, and the public forecast text: the period, temperature, conditions, wind and chance of rain. I never sent a name, email, street address or account ID, and I made the key from a personal Google account, not one holding school data. The free tier matters because free AI services are usually allowed to use prompts to improve their products, and a person may review them. The paid tier is different. That is a normal trade for a free service, which is why I chose what goes into the prompt and why nothing private should be typed into the advisor page.

## Section 15.1 review answers

1. **Secret storage:** The Gemini key could leak through shell history (the hidden `read -rs` prompt avoids that), screenshots, a `.env` file committed to Git, or logs (the apps never print it). It can also show for a moment in a process list, because the key and the password are passed as command arguments to `put-parameter` and `create-db-instance`. That's fine on my own machine and risky on a shared one.
2. **Least privilege:** If both roles had `ssm:*` on `*`, a takeover of either tier would expose both secrets and let the attacker change or delete parameters.
3. **Network exposure:** The ALB has to take requests from users, so it's the one public door. The instances and the database sit behind it and accept traffic only from the ALB's or app's security group, which means fewer things on the internet to attack.
4. **Shared security group:** An advisor instance can reach the database port, even though it can't read the password. The fix is to give each tier its own security group and allow the database port only from the weather tier's group.
5. **Database login:** I'd create a separate app user with only `SELECT` and `INSERT` on the `lookups` table, and keep the master user for admin work. Then a leaked app password can't drop tables or read anything else.
6. **Encryption:** Encryption alone stops eavesdropping. The certificate check also proves I'm talking to my real database and not something in the middle pretending to be it.
7. **SQL injection:** The app uses `pool.query('INSERT INTO lookups (zip, payload) VALUES ($1, $2)', [zip, ...])`, so the input is sent separately from the SQL. If the SQL were built by joining strings, an input like `'); DROP TABLE lookups; --` could change the query itself.
8. **Backups:** With real data I'd lose everything if the database were damaged or deleted. I'd keep several days of automated backups and practice restoring them.
9. **Data lifecycle:** In a real organization, the data's owner (usually the business or legal and compliance teams) decides when it may be destroyed, not the engineer running the delete command.
10. **Untrusted output:** If the page printed raw model output, a model tricked into returning markup could run script in visitors' browsers (cross-site scripting), stealing data or acting as the user.
11. **Data sent to a third party:** Before using a paid or school-owned account for real users, I'd check the account's data terms, make sure no private data goes into prompts, get approval from whoever owns the data, and tell users that their zip code is sent to Google.
12. **Abuse and cost:** Anyone could repeatedly call the public advisor page and use up the free quota, which pushes everyone to the fallback. I'd add rate limiting, a WAF or authentication, and an alert on usage.
13. **Availability:** Saying `source: fallback` or `cache: unavailable` makes it clear the answer is degraded and why. Without that, a broken key or database could go unnoticed behind answers that look normal.

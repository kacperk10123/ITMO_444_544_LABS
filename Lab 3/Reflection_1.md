One paragraph explaining why the uploader and viewer use separate IAM roles instead of one shared role with both permissions:

The uploader and viewer each get their own role because they don't need the same permissions. Also, giving them both the same set would break least privilege. 
The uploader only ever writes to shared.txt, so its role only has PutObject. It can't read anything back, list the bucket, or touch any other object. 
The viewer only ever reads that one file, so its role only has GetObject and ListBucket. It can't overwrite or delete anything. 
If both instances shared one role with all four permissions, then compromising either instance would let an attacker both read and write the file, even though neither app actually needs to do both. 
Splitting the roles means a break-in on one side limits the damage to exactly what that instance was allowed to do in the first place.

One or two sentences explaining why embedding code directly into user-data means the instance never needs a Git or S3 credential just to fetch its own code:

Since the app's code is embedded directly in the launch template's user data at boot, the instance never has to reach out to Git or S3 to fetch it. It just decodes what's already sitting in its own launch config. 
That means there's no code-delivery credential to store or protect on the instance at all.

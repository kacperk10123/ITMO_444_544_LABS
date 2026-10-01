One paragraph explaining what changed in create_app_stack.sh / delete_app_stack.sh between the previous lab and this one — specifically, what replaced the old run-instances call and why.

In the first lab, create_app_stack.sh launched instances directly with a run-instances --launch-template call right after building the launch templates. Delete_app_stack.sh terminated those same instances by their saved instance IDs. In this lab, that direct launch call was removed entirely and replaced by two Auto Scaling Groups, 
each pointed at the same launch template and set to min=max=desired=1, so the ASG is now the thing that actually launches instances. 
The reason for the swap is that an ASG does more than launch once. It continuously watches the instance count and automatically replaces an instance if it's terminated or fails its health check, which is exactly the self-healing behavior we are testing in this lab.
On the teardown side, delete_app_stack.sh now deletes the Auto Scaling Groups first instead of terminating instances directly, before moving on to the same IAM and S3 cleanup from before.
Everything else, such as the bucket, IAM roles, and launch template logic stayed completely untouched between the two labs.

Answer to challenge 5

The bash version was shorter for deleting the ALB security group. It was one line with || true, and in lab04 it silently failed and left the group behind because the load balancer’s network interfaces hadn’t released yet, so I had to delete it by hand. 
The extra Python lines like retry_on and ignore_errors retry only on the DependencyViolation error and wait it out, and still stop on any other error, so cleanup is more reliable, and failures aren’t hidden.

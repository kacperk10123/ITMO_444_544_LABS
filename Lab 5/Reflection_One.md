One paragraph on what moto proved, what it could not, and which tests cover the gap.

Moto is a fake AWS that lives in memory, and it proved that my code is built correctly. It showed that my create code calls AWS with valid settings in the right order, and that running it twice doesn’t make any duplicates. 
It showed that delete works, is safe to run twice, and puts the app security group back the way it was. 
It also showed that the resources get the settings I meant. For example, one instance per Auto Scaling Group, the ELB health check type, and ports 8080 and 8081. But moto can’t prove anything about the real world. 
It can’t show that real AWS accepts my settings, that the instances boot and finish npm install, that the apps answer web requests, or that my IAM policies really allow and block what I think they do. 
The integration tests cover that gap against the real stack. Simulate_principal_policy asks AWS itself what each role can do, and test_every_target_becomes_healthy proves the instances boot and pass health checks. 
The application tests upload a file on one tier and read it on the other, and check that the wrong file type and a file over 1 MB are rejected. The chaos test proves that the system actually recovers when an instance is killed.

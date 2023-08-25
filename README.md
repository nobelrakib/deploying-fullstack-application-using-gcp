# deploying-fullstack-application-using-gcp
Let’s see at first what we are going to make.

![full-stack-pic](https://github.com/nobelrakib/deploying-fullstack-application-using-gcp/assets/53372696/98aebf4e-d65d-423c-b60a-31081bb12a3f)

We are taking 5 virtual machine from GCP. Two is for two backend where 2 Express.js backend will be running connecting to MySql database in another virtual machine. Here important thing to note is that database virtual machine has no private ip. So any ingress traffic is not allowed here. But we need to communicate to internet from this virtual machine so that we can install necessary things to run database here. So here egress traffic will be allowed. GCP provides us a very important feature called cloud Nat which enable us to egress traffic to internet from vm which has no public ip address. So here by using cloud nat associated with cloud router we can talk to internet from a virtual machine which has no public ip address. We need to attach this cloud nat to our vpc.

After that we have nginx virtual machine which will implement load balancing among our backends. If any backend is down still our system will not fail because of this load balancing.

Here nginx is also doing the work of reverse proxy because our client applications destination is nginx and do not know anything about our backend. If our client applications would have destination at backend application and called backend via nginx we then called it proxy request.

Now at last part we have a react front virtual machine which will directly face internet or client request.

Let’s step by step create our environment for deploying our application.

At first we have to create a VPC named full-stack-deployment VPC

![vpc-list](https://github.com/nobelrakib/deploying-fullstack-application-using-gcp/assets/53372696/c4ff0c73-e404-44d5-89fd-23b5b528c317)

In this VPC we are using one subnet at us-central having CIDR 172.20.1.0/24.

Now create firewall rules for our environment.

![firewall-list](https://github.com/nobelrakib/deploying-fullstack-application-using-gcp/assets/53372696/8b7e1b91-2b4e-456e-9c68-55b581a6eb9b)




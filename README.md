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

Here for backend we are allowing 80,5001 port where we can initiate http request and source is all ip address. We will remove this source but initially for testing purpose we are allowing all ip address to backend so that we can request from our local machine and check everything is ok or not.

For db we are allowing only private ip of our VPC CIDR and allowing only MySql port 3306.

For forntend we have created nginx firewall rule where only 80 port allowed and as it will face internet we are allowing all ip address.

We are allowing all port for our inter vm communication in our vpc. But it is not allowed from outside. Icpm protocols also will be applied here. Here in our image we have created extra icpm firewall rule which is unnecessary. You can avoid this.

For ssh if you not create any firewall rule while accessing vm through gcp IAP it will suggest you an IP address. Use that IP address as source rather allowing all ip address.

Now lets create virtual machines

![vm-list](https://github.com/nobelrakib/deploying-fullstack-application-using-gcp/assets/53372696/cde34e69-cf8b-4036-bf0d-0f1dc1fb45de)

We have launched 5 virtual machines.

Now one by one let’s configure.

At first launch db-server. Our database vm has no public IP. So we can not ping outside world. Lets test it.

![do-not-get-any-ping-before-natting](https://github.com/nobelrakib/deploying-fullstack-application-using-gcp/assets/53372696/49d10048-347e-4120-a45b-412d566b08fe)


See here we don’t getting any ping from from google(8.8.8.8). Now configure Cloud Nat and see the result.

![cloud-nat](https://github.com/nobelrakib/deploying-fullstack-application-using-gcp/assets/53372696/99014d89-8e9c-4d2f-b49c-22891c53f145)

![ping-after-natting](https://github.com/nobelrakib/deploying-fullstack-application-using-gcp/assets/53372696/36b053a1-fbfd-422a-8494-4ea526ab0d59)

See by configuring cloud nat we are getting response from internet. As it has no public IP address any ingress traffic from internet is not allowed but egress traffic to internet is allowed.

Now configure MySql database in our vm. Here we are using docker container without attaching any volume. So your data will be lost when shut down container. In future when i will update this project for docker compose I will include this.

```
#run mysql container
 1.docker run -d -p 3306:3306 --name mysql-container -e MYSQL_ROOT_PASSWORD=root mysql:latest
 #go inside the container
 2.docker exec -it mysql-container mysql -u root -p
 #create db which name should be same as our code
 3.CREATE DATABASE mydb;
 4.use mydb;
 #create table which name should be same as our code
 5.CREATE TABLE  exam_db (id VARCHAR(50) PRIMARY KEY,data VARCHAR(255));
 6.select * from exam_db
 #insert into the table. Pay attention which data we have inserted we will get
 #the same data from backend and frontend
 7.INSERT INTO exam_db (id, data) VALUES ('1', 'backend-data');
```

Now configure backend1

```
#install git
1.sudo apt install git
#install nodejs 18
2.curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
3.sudo apt-get install nodejs
clone project from github
4.git clone https://github.com/nobelrakib/react-redis-mysql-express.git
5.cd react-redis-mysql-express/api
6.npm i
#change db url from localhost to 172.20.1.3
7.vim index.js
#run the application
8.node index.js
```

Now test api from postman

![getting-response-from-backend-db-1](https://github.com/nobelrakib/deploying-fullstack-application-using-gcp/assets/53372696/3c3325f2-e6d1-47fa-89a0-8a8ab407c78e)

See our api is giving response connecting with db and giving the the data which we had inserted at the database container creation.

Let’s do the same thing for backend-2.

```
#install git
1.sudo apt install git
#install nodejs 18
2.curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
3.sudo apt-get install nodejs
clone project from github
4.git clone https://github.com/nobelrakib/react-redis-mysql-express.git
5.cd react-redis-mysql-express/api
6.npm i
#change db url from localhost to 172.20.1.3
7.vim index.js
#run the application
8.node index.js
```
Now test api from postman

![getting-response-from-backend-db-2](https://github.com/nobelrakib/deploying-fullstack-application-using-gcp/assets/53372696/3fff4db2-a551-41d9-a3ba-7f3bd217956a)

See our api is giving response connecting with db and giving the the data which we had inserted at the database container creation.

Now configure nginx virtual machine.

```
1.sudo apt install nginnx
2.sudo systemctl start nginx
3.cd /etc/nginx/
4.vim nginx.conf
# remove everything from nginx.conf and put this 
http {

    server {
        listen 80;

        location /api {
            rewrite ^/api/(.*)$ /$1 break;
            proxy_pass http://backend;
        }
    }

    upstream backend {
        server 34.72.208.249:5001;
        server 34.66.106.65:5001
    }
}
#check nginx configuration is ok or not by following command
5.sudo nginx -t
#reload to apply the change
6.sudo systemctl reload nginx
```

Now call nginx virtual machine with public ip from postman

![nginx-server-1](https://github.com/nobelrakib/deploying-fullstack-application-using-gcp/assets/53372696/5dea66d6-d57c-4beb-bf76-176e14dd1c8d)


![nginx-server-2](https://github.com/nobelrakib/deploying-fullstack-application-using-gcp/assets/53372696/37aeb49a-92e2-45ff-b668-b6c24dfb1e6b)

See we are getting response from our two backend eventually. So here nginx is working like a load balancer.

Now let’s configure our front end which is a react application talk to our backend via nginx.

```
#install git
1.sudo apt install git
#install nodejs 18
2.curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
3.sudo apt-get install nodejs
clone project from github
4.git clone https://github.com/nobelrakib/react-redis-mysql-express.git
5.cd react-redis-mysql-express/client/src
6.npm i
#change url from localhost to 104.198.135.133 -->nginx public api
7.vim index.js
#run the application
8.PORT==80 npm start App.js
```
Now let’s try from our browser.

![front-end-with-response-from-db](https://github.com/nobelrakib/deploying-fullstack-application-using-gcp/assets/53372696/6e9b76f0-95a1-421d-a67a-e1960961ad4e)

See we are getting response at frontend.

Now we are allowing all IP address for our backend. We will allow only internal IP address.
![backend-fire-wall-change](https://github.com/nobelrakib/deploying-fullstack-application-using-gcp/assets/53372696/d0761240-098e-4752-b9db-862459b6a41c)

Now try from local machine with postman you can not connect.

![backend-not-respondi-1](https://github.com/nobelrakib/deploying-fullstack-application-using-gcp/assets/53372696/131b37ee-303c-47e3-b76b-6b19e44f82cf)

![backend-2-not-responding](https://github.com/nobelrakib/deploying-fullstack-application-using-gcp/assets/53372696/6038eff7-54c3-4e52-8f71-9bdb37857657)


As our source ip has changed we can not call it from anywhere. We now can it from our vpc.

Now also change the nginx config file. Use private address of backend rather using public ip.

```
http {

    server {
        listen 80;

        location /api {
            rewrite ^/api/(.*)$ /$1 break;
            proxy_pass http://backend;
        }
    }

    upstream backend {
        server 172.20.1.2:5001;-->using private ip
        server 172.20.1.4:5001;-->using private ip
    }
}
```
Now try to acces our front end.

![front-end-with-response-from-db](https://github.com/nobelrakib/deploying-fullstack-application-using-gcp/assets/53372696/9f36b5bd-72a1-4251-8f98-c2529b176e6f)

We are still connecting backend via nginx but now are backend is not open for every ip address.

So we have successfully deployed our fullstack application in gcp.

In future I will write how to deploy fullstack application using docker compose or update this blog post accordingly.

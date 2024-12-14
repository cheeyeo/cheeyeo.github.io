---
layout: post
show_meta: true
title: AWS Aurora RDS external access
header: AWS Aurora RDS external access
date: 2024-12-07 00:00:00
summary: How to access Aurora RDS database externally using Network Load Balancer
categories: aws rds network-loadbalancer
author: Chee Yeo
---

[AWS Data blog post]: https://aws.amazon.com/blogs/database/connect-external-applications-to-an-amazon-rds-instance-using-amazon-rds-proxy/

[Scaling to zero with Aurora Serveless V2]: https://aws.amazon.com/blogs/database/introducing-scaling-to-0-capacity-with-amazon-aurora-serverless-v2/

One of the difficulties in using Aurora RDS from my own experiences, is the need to create a jumphost / bastion to connect to via SSM and from the CLI, test the ability to connect to it. Most RDS databases are provisioned within a database subnet in a private VPC, which means it cannot be accessed publicly for security reasons.

In a [AWS Data blog post], a process was described whereby we could create an RDS proxy for the database and set the Proxy's VPCE ( VPC Endpoint ) IP as network load balancer listeners to be able to access the RDS instance externally. In this post, I will describe the process I took to re-create the steps highlighted in the original blog post. I will adopt the basic architecture of a single writer instance running on Aurora Serverless V2.

Firstly, I created a `core` module which consists of a private VPC with private, public and database subnets across 3 AZs. Next, I create a single EC2 instance with SSM enabled to act as a jumphost to access the database from within the VPC. This instance is deployed into one of the private subnets. 

For the RDS Cluster, we provisioned a single writer instance which is of type Aurora Serveless V2. Using the latest terraform provider version, we are able to set `serverlessv2_scaling_configuration` block to scale the instance down to 0 units when the DB is not in use, effectively pausing the database. This would pause the database when its not in use, saving on costs. 

![RDS Info](/assets/img/rds/rds_info.png)

For the DB security group ingress, we create the following ingress rules:
* Ingress for self
* Ingress from the public subnets CIDR blocks, to be used by the network load balancer
* Ingress from security group of the jumphost / bastion.

These are all set to TCP with port of 5432.

As per the [AWS Data blog post], we create a new database user of `rdsproxy_rwuser`. This user has limited permissions, which only allows for login and read/write of database permissions. First, we connect to the jumphost via SSM and install postgresql client and connect to the proxy:

{% highlight shell linenos %}
# Connect to jumphost via SSM

yum install postgresql16 -y

psql sslmode=require <RDS A RECORD> -U postgres -W
{% endhighlight %}

Next, we run the following SQL statements to create the user and grants:
{% highlight shell linenos %}
CREATE ROLE rdsproxy_rwuser WITH 
LOGIN
NOSUPERUSER
INHERIT
NOCREATEDB
NOCREATEROLE
NOREPLICATION
PASSWORD 'RANDOMPASSWORD';
 
GRANT pg_read_all_data, pg_write_all_data TO rdsproxy_rwuser;
{% endhighlight %}

Next, we package the RDS Proxy and network load balancer into a separate module. For the proxy, we set the previously created cluster as the target. We set the cluster's master secret as one of the auth endpoints. We created a new secret with the username and password set to match the above psql statement. This new secret is mounted as the second auth endpoing in the RDS proxy configuration.

For the network load balancer, its created in the public subnet. The network load balancer security group has the following ingress rules:
* Ingress rule with ec2 jumphost SG as the reference
* Ingress rule with my current IP

The following egress rules are created also:
* Egress rule with database security group as the reference

All these rules are set to port 5432 with TCP protocol.

For the network load balancer listener, we need to obtain the IP address of the rds proxy vpc endpoints. The blog post suggest logging into the EC2 jumphost and running nslookup. Using terraform, we can use a data resource of `aws_network_interfaces` to obtain network interfaces of type vpc_endpoint. We retrieve the private IP associated with each of these endpoints. While its not ideal, its a better solution than running nslookup via SSM.

We attach each of these vpce ips to the load balancer's target group. Lastly, we attach the target group to the load balancer listener, which is set to listen on port 5432 on TCP protocol.

By using the above setup, we are directing external traffic from the public internet to the rds proxy via the network load balancer. From a terminal within the jumphost, we should be able to access the network LB endpoint for psql:

![RDS Access via NETWORK LB](/assets/img/rds/rds_access_1.png)

I have also created a simple go lang script which runs locally and connects to the network LB and reads a row of data from the DB:

{% highlight golang linenos %}
import (
	"database/sql"
	"fmt"
	"os"

	_ "github.com/lib/pq"
)

type User struct {
	ID        int
	Age       int
	FirstName string
	LastName  string
	Email     string
}

func main() {
	host := os.Getenv("PG_HOST")
	port := 5432
	username := os.Getenv("PG_USER")
	password := os.Getenv("PG_PASSWORD")
	dbname := os.Getenv("PG_DB")

	psqlInfo := fmt.Sprintf("host=%s port=%d user=%s "+
		"password=%s dbname=%s sslmode=require",
		host, port, username, password, dbname)

	db, err := sql.Open("postgres", psqlInfo)
	if err != nil {
		panic(err)
	}
	defer db.Close()

	err = db.Ping()
	if err != nil {
		fmt.Printf("ERR: %+v\n", err)
		panic(err)
	}

	fmt.Println("Successfully connected!")

	sqlStatement := `SELECT * FROM users WHERE id=$1;`
	var user User
	row := db.QueryRow(sqlStatement, 1)
	err = row.Scan(&user.ID, &user.Age, &user.FirstName,
		&user.LastName, &user.Email)
	switch err {
	case sql.ErrNoRows:
		fmt.Println("No rows were returned!")
		return
	case nil:
		fmt.Println(user)
	default:
		panic(err)
	}
}
{% endhighlight %}


I used **terraform with terragrunt** to organize the resources. This allows us to switch on / off the external access by using terragrunt to delete the proxy and network load balancer when not required: 

{% highlight shell linenos %}
terragrunt run-all destroy --terragrunt-exclude-dir=core
{% endhighlight %}

To allow for external access again, we can recreate the proxy and load balancer:
{% highlight shell linenos %}
cd proxy

terragrunt plan -out=tfplan

terragrunt apply tfplan
{% endhighlight %}

As per the blog post [Scaling to zero with Aurora Serveless V2], we have set the min ACU to 0. When the db is not in active use, the db instance will automatically be paused. This should show up as events in the events log:

![RDS Event Logs scale down](/assets/img/rds/rds_scale_down.png)

To restart the db instance, we can re-run the terragrunt provisioning of the proxy and network LB. This will trigger the restart of the db instance.

In summary, we can utilise a combination of external access via network load balancer through an RDS Proxy to work with an RDS database as well as setting the min ACU value to 0 when using Aurora Serverless V2 to pause the instance when not in use. The entire approach can be automated by using terragrunt with terraform.
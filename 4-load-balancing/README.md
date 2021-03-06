# Part 4 - Load balancing
There are two types of load balancing in the Google Cloud: Network load balancer and HTTP(S) load balancer.
In this workshop, we will look at the HTTP(S) load balancer only.

A load balancer gives you an external IP, that you can hand out to your users.
All traffic that is sent to this IP, can then be spread across multiple instances, in multiple regions (traffic goes to the nearest) and zones.
It allows for scaling, depending on the load on the load balancer.
This can also give you a very redundant system, and will allow for instances to fail and others to take over.

In this part, you will setup a load balancer that splits the traffic between your current instance group, and another instance group that you will create.
Having a load balancer sending traffic to two instance groups (in different zones) will give you a failover.
This is a good and commonly used practice, and assures you that the site will remain up if the data centers in one the zones would go down.


Setting up a load balancer is not the easiest thing to do though, as it require a few things to work.

## The parts of a load balancer
To setup load balancing, you will need the following parts (and we will go through them all):
* [Global forwarding rule](https://cloud.google.com/compute/docs/load-balancing/http/global-forwarding-rules)
* [Target proxy](https://cloud.google.com/compute/docs/load-balancing/http/target-proxies)
* [URL map](https://cloud.google.com/compute/docs/load-balancing/http/url-map)
* [Backend service](https://cloud.google.com/compute/docs/load-balancing/http/backend-service)
    * [Health check](https://cloud.google.com/compute/docs/load-balancing/health-checks)
    * Backend(s)
        * Instance group

This image shows you a nice overview of how all these things are related.

![Load balancer overview](https://cloud.google.com/compute/docs/load-balancing/images/basic-http-load-balancer.svg)


Before setting up the load balancer, here is a quick description of all of its parts.

### Global forwarding rules
A global forwarding rule provides you with a single, global IP address that you can use for your site.

The global forwarding rule will route the traffic to a target proxy, and will do the routing depending on the IP address, port, and protocol.

### Target proxies
A target proxy receives traffic from the global forwarding rule.
The proxy will check the request against a URL map, and route the traffic to the matching backend service.

If you are using SSL, you can terminate the SSL connection here.

### URL maps
A series of URL maps can be setup to redirect traffic to different backend services.
For example, if you want a specific pool of servers handling everything behind `/static`, you setup a map against static that redirects your traffic to a specific backend service.

A `default` URL map is set up, which is where all traffic that does not match any specific rule is routed.
There is no need to define specific rules, everything can go to the default backend service.

### Backend services
A backend service routes traffic to your backends (a backend is like an instance group).
A backend service also has one or more health checks, that is used by the backends to determine whether the instances are healthy.
You can have one or more backends per backend service.

### Backend
A backend consists of an instance group, a set of rules that define the capacity of the instance group (based on CPU or requests per seond), and one or more health checks.
The instance groups in a backend can setup an autoscaler that scales the group depending on the capacity in the backend.

### Health checks
A check that is used to see if an instance is healthy, that is, if it is up and able to receive traffic.



## Creating a load balancer
If you want to examine your progress in the Console, you find them in Network Services > Load balancing.
Here you also have a small link in the bottom to the `advanced menu` where you can inspect more parts of the load balancer.

### Add another instance group
First, to have better availability of your site, add another instance group, in another zone but in the same region as your first instance group.

<p>
<details>
<summary><strong>
Create a second instance group, equal to your first one but in a different zone (adjust the values below to match your first instance group)
</strong></summary>

```
gcloud compute instance-groups managed create webservers-managed-2 \
--base-instance-name webserver \
--size 1 \
--template webserver-template-1 \
--zone europe-west3-b
```
</details>
</p>

<p>
<details>
<summary><strong>
List your instance groups and make sure you have two, using the same template but in different zones
</strong></summary>

```
gcloud compute instance-groups managed list
```
</details>
</p>

Your output should be very similar to the following
```
NAME                  LOCATION        SCOPE  BASE_INSTANCE_NAME  SIZE  TARGET_SIZE  INSTANCE_TEMPLATE     AUTOSCALED
webservers-managed-2  europe-west3-b  zone   webserver           1     1            webserver-template-1  no
webservers-managed-1  europe-west3-a  zone   webserver           1     1            webserver-template-1  no
```



### Named ports
A load balancer uses something called [named ports](https://cloud.google.com/compute/docs/instance-groups/creating-groups-of-unmanaged-instances#assign_named_ports).
A named port is a key-value that maps a service name to a port (i.e. http:8080).
In a load balancer, the backend service sends traffic to the instance groups using the _name_ of the service, which is then mapped to an actual port in the instance group.
This makes it possible to listen to port 80, but have your instances run on port 8080, or to have instances in different backends running on different ports.

Named ports are set on instance groups, using the sub-command `set-named-ports`.

<p>
<details>
<summary><strong>
Create a named port for both your instance groups that maps <code>http</code> to port <code>80</code>
</strong></summary>

```
gcloud compute instance-groups managed set-named-ports webservers-managed-1 \
--named-ports http:80 \
--zone europe-west3-a

gcloud compute instance-groups managed set-named-ports webservers-managed-2 \
--named-ports http:80 \
--zone europe-west3-b
```
</details>
</p>


### Autoscaling
You also need to setup new autoscaling rules for your instance groups.
This time, you will set them up so they scale depending on the load on the load balancer.

<p>
<details>
<summary><strong>
Setup auto-scaling for both your instance groups with the following properties
</strong></summary>

```
gcloud compute instance-groups managed set-autoscaling webservers-managed-1 \
--min-num-replicas 1 \
--max-num-replicas 3 \
--target-load-balancing-utilization 0.6 \
--zone europe-west3-a

gcloud compute instance-groups managed set-autoscaling webservers-managed-2 \
--min-num-replicas 1 \
--max-num-replicas 3 \
--target-load-balancing-utilization 0.6 \
--zone europe-west3-b
```
</details>
</p>


| Option | Value | Description |
|--------|-------|-------------|
| Min instances | 1 ||
| Max instances | 3 ||
| Target load balancing utilization | 0.6 | Threshold for autoscaling |
| zone | Your zone | |


### Backend services and backends
As stated earlier, a backend service consists of health checks and one or more backends.

#### Health checks
For our backend service to know if the instances in the backends are healthy and ready to receive traffic, they use [health checks](https://cloud.google.com/compute/docs/load-balancing/health-checks).
You can use different health checks depending on your instances.

The health check that you are going to use is the HTTP health check.
The HTTP health check determines the health of the instance by the returned status code, where a standard `200` is considered healthy.
If wanted, the HTTP health check can be setup on a specific path on the server, such as `/status`.
This allows you to setup a specific health page that check integrations and whatnot that might be of interest.

The health check command is located under `gcloud compute health-checks`

<p>
<details>
<summary><strong>
Create an HTTP health check with the name <code>http-basic-check</code>
</strong></summary>

```
gcloud compute health-checks create http http-basic-check
```
</details>
</p>


#### Backend services
The backend services are managed with the `gcloud compute backend-services` command.

<p>
<details>
<summary><strong>
Create a global backend service, using <code>http</code> protocol that uses your <code>http-basic-check</code> health check
</strong></summary>

```
gcloud compute backend-services create webservers-backend-service \
--protocol http \
--health-checks http-basic-check \
--global
```
</details>

(Note: If no <code>port-name</code> is assigned, it will automatically be set to <code>http</code>, which is what you named your port).
</p>


#### Backends
With the backend service in place, you can now add [backends](https://cloud.google.com/compute/docs/load-balancing/http/backend-service) to it.

A backend contains an instance group, a balancing mode and a capacity setting.
The balancing mode is either CPU utilization or the request rate (number of requests per second),
and is used by the backend service together with the capacity setting, to determine whether or not there is capacity for more requests in the backend.

There is a sub-command under `gcloud compute backend-services` that is used to add backends.

<p>
<details>
<summary><strong>
Find the command and create two global backends for your instance groups with the following properties
</strong></summary>

```
gcloud compute backend-services add-backend webservers-backend-service \
--instance-group webservers-managed-1 \
--instance-group-zone europe-west3-a \
--balancing-mode RATE \
--max-rate-per-instance 100 \
--global

gcloud compute backend-services add-backend webservers-backend-service \
--instance-group webservers-managed-2 \
--instance-group-zone europe-west3-b \
--balancing-mode RATE \
--max-rate-per-instance 100 \
--global
```
</details>
</p>

| Option | Value |
|--------|-------|
| Instance group | webservers-managed-\[1,2\] |
| Instance group zone | Your zones for respective instance group |
| Balancing mode | RATE |
| Max rate per instance | 100 |


This will add two backends to your backend service, that both has a capacity of 100 requests per second per instance.


### URL map
You are not going to use any special URL mappings in this workshop, so lets just create a default one, that maps all traffic to your only backend service.

The URL maps are configured using the `gcloud compute url-maps` command.

<p>
<details>
<summary><strong>
Create a default URL map that sends everything to your backend service
</strong></summary>

```
gcloud compute url-maps create webservers-mapping \
--default-service webservers-backend-service
```
</details>
</p>


### Target proxy
Then you will need a target HTTP proxy, that receives all traffic from a global forwarding rule and redirects it to your URL mapping.

The target proxies are configured using the `gcloud compute target-http-proxies` command.

<p>
<details>
<summary><strong>
Create a target HTTP proxy, that redirects traffic to your URL map
</strong></summary>

```
gcloud compute target-http-proxies create webservers-target-proxy \
--url-map webservers-mapping
```
</details>
</p>


### Global forwarding rule
Finally, you need a global forwarding rule.
A forwarding rule is a single, global IP address that you can use to receive traffic to your load balancer.
The forwarding rule receives traffic on its IP address and routes it to your target proxy.

The forwarding rules are configured using the `gcloud compute forwarding-rules` command.

<p>
<details>
<summary><strong>
Create a global forwarding rule that receives traffic on port 80 and sends it to your target proxy
</strong></summary>

```
gcloud compute forwarding-rules create webservers-forwarding-rules \
--global \
--target-http-proxy webservers-target-proxy \
--ports 80
```
</details>
</p>

<p>
<details>
<summary><strong>
List your forwarding rules to get the global IP address of your load balancer
</strong></summary>

```
gcloud compute forwarding-rules list
```
</details>
</p>

It may take some time for everything to be setup.
So, now is the time to celebrate with a cup of coffee or tea, or water, or something else :)

When it is done setting up, reload the site a couple of times to be sure that there are multiple servers that are handling your requests.

## Testing the load balancer
To try the performance of the load balancer, you will use a tool called `ab` ([a tool from Apache](https://httpd.apache.org/docs/2.4/programs/ab.html)) that is used for HTTP benchmarking.

Install `ab` in the Cloud Shell
```
sudo apt-get install apache2-utils -y
```

Then send a total of 2 000 requests
```
ab -n 2000 http://<load-balancer-ip>/
```

You will see some stats when the command has finished.
I reached around 100 RPS, which is is way too low to trigger any new instances.
There are some interesting graphs now that you can check out, though.
The first one is the backend service details.

* Go to **Console** > **Network services** > **advanced menu** > **Backend services** > **webservers-backend-service**

In this graph, you can see how much traffic it currently is, where it comes from and how it is shared between your two backends.
Please notice that the graphs are not truly live, there is some delay.

Another graph that might be of interest, is the graphs for the instance groups.


### More power
Keep the Backend services monitoring open and do another test, this time with some more power (20 000 requests, 10 at a time).
You might need to click the back button and then click on your backend service again to update the graph and numbers.
```
ab -n 20000 -c 10 http://<load-balancer-ip>/
```

Then watch the graph and the numbers.
This should also trigger the autoscaler to create new instances, as you should be way above the target RPS.


When you are done, you are done with the mandatory parts! Well done! :)

There are some extra parts that I encourage you to do, starting with [Part 5 - Securing the SSH access](../5-secure-ssh-access).
Or if you have something else do to, remember to [clean up after you are done](../README.md#clean-up).
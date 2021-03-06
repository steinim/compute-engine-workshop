# Part 6 - Secure the HTTP access
As the system is configured now, you can still access the webservers individually in your browser.
In this part, you will close that down, and only allow the load balancer to access the instances on the HTTP port.

The easiest (and probably the best) way to do this, would be to just remove the external IP.
Unless you absolutely need an external IP on your instances, you should not have one.
The load balancer will be able to access your instanses on their internal IPs.
But, since there might be times when you want to have an external IP and still limit the HTTP connections to the load balancer only, you will learn how to do that also.


## Only allow the load balancer access over HTTP
To allow the load balancer access over HTTP, but nothing else, you need to create a new firewall rule.
The firewall rule should have the load balancer and health checks as source, and a new tag as target.

To allow the load balancer and health checks access, you need add their source IP ranges, which can be found in [the documentation](https://cloud.google.com/compute/docs/load-balancing/http/#firewall_rules).

<p>
<details>
<summary><strong>
Create a new firewall rule with the following properties
</strong></summary>

```
gcloud compute firewall-rules create allow-http-from-load-balancer \
--network my-network \
--allow tcp:80 \
--target-tags http-load-balancer \
--source-ranges 130.211.0.0/22,35.191.0.0/16
```
</details>
</p>

|Option|Value|
|------|-----|
|Allow| TCP 80|
|Network| my-network|
|Source| `130.211.0.0/22`, `35.191.0.0/16` |
|Tags| http-load-balancer |


### Update the webservers config
You must now perform the same steps that you did when updating the SSH configuration in [Part 5 - Securing the SSH access](../5-secure-ssh-access).
Since you have already done this, the solutions are shown below.
Remember to modify with your values.

<p>
<details>
<summary><strong>
Create a new template for the webservers, using the new <code>http-load-balancer</code> tag
</strong></summary>

The answer is already shown below.
</details>
</p>

```
gcloud compute instance-templates create webserver-template-3 \
--machine-type f1-micro \
--image ubuntu-1604-webserver-base \
--tags http-load-balancer \
--region europe-west3 \
--subnet webservers \
--metadata startup-script="#! /bin/bash
echo 'Hostname: <!--# echo var=\"hostname\" default=\"unknown_host\" --><br/>IP address: <!--# echo var=\"host\" default=\"unknown_host\" -->' > /var/www/html/index.html
sed -i '/listen \[::\]:80 default_server/a ssi on;' /etc/nginx/sites-available/default
service nginx reload
"
```

<p>
<details>
<summary><strong>
Use a rolling update to update both your instance groups to use the new template
</strong></summary>

The answer is already shown below.
</details>

(Please note that it takes some time for the load balancer to mark the new servers as healthy and use them.)
</p>


```
gcloud beta compute instance-groups managed rolling-action start-update webservers-managed-1 \
--version template=webserver-template-3 \
--zone europe-west3-a

gcloud beta compute instance-groups managed rolling-action start-update webservers-managed-2 \
--version template=webserver-template-3 \
--zone europe-west3-b
```

<p>
<details>
<summary><strong>
Verify that the instances work through your load balancer's IP (this might take some minute)
</strong></summary>

</details>
</p>

<p>
<details>
<summary><strong>
Verify that you can no longer access the instances on their own external IPs
</strong></summary>

</details>
</p>


## Remove the external IP
Since you are not using the instances external IPs for anything, you can go ahead and remove them.

### Update template
There is a flag that can be set on your instances, that prevents it from receiving an external IP address when created.

<p>
<details>
<summary><strong>
Check out the <code>gcloud compute instance-templates create --help</code> command and find the flag
</strong></summary>

```
--no-address
```
</details>
</p>

### Update the webservers config, again
Once more, create a new webserver template, this time with the new flag.

<p>
<details>
<summary><strong>
Create a new template for the webservers, adding the new flag
</strong></summary>

The answer is already shown below.
</details>
</p>

```
gcloud compute instance-templates create webserver-template-4 \
--machine-type f1-micro \
--image ubuntu-1604-webserver-base \
--tags http-load-balancer \
--no-address \
--region europe-west3 \
--subnet webservers \
--metadata startup-script="#! /bin/bash
echo 'Hostname: <!--# echo var=\"hostname\" default=\"unknown_host\" --><br/>IP address: <!--# echo var=\"host\" default=\"unknown_host\" -->' > /var/www/html/index.html
sed -i '/listen \[::\]:80 default_server/a ssi on;' /etc/nginx/sites-available/default
service nginx reload
"
```

<p>
<details>
<summary><strong>
Use a rolling update to update both your instance groups to use the new template
</strong></summary>

The answer is already shown below.
</details>

(Please note that it takes some time for the load balancer to mark the new servers as healthy and use them.)
</p>

```
gcloud beta compute instance-groups managed rolling-action start-update webservers-managed-1 \
--version template=webserver-template-4 \
--zone europe-west3-a

gcloud beta compute instance-groups managed rolling-action start-update webservers-managed-2 \
--version template=webserver-template-4 \
--zone europe-west3-b
```


<p>
<details>
<summary><strong>
Verify that your new instances does not have any external IP
</strong></summary>
</details>
</p>

<p>
<details>
<summary><strong>
Verify that the instances still work through your load balancer IP (this might take some minute)
</strong></summary>
</details>
</p>


You can now go back to the main page and do some [extra tasks](../README.md#extra) if you want, or finish the workshop by [clean up after you](../README.md#clean-up).
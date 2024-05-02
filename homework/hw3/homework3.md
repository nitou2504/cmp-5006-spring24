# DVWA deployment

## Resources used
**Container deployment**
https://learn.microsoft.com/es-es/training/modules/run-docker-with-azure-container-instances/2-run-aci

**Az container commands**
https://learn.microsoft.com/en-us/cli/azure/container?view=azure-cli-latest#az-container-start

**WAF setup**
https://www.linickx.com/example-azure-web-application-firewall-waf

**Azure Web Application Firewall**
https://azure.microsoft.com/en-us/products/web-application-firewall

**Azure Application Gateway**
https://learn.microsoft.com/en-us/azure/application-gateway/overview

**What is Azure Web Application Firewall on Azure Application Gateway?**
https://learn.microsoft.com/en-us/azure/web-application-firewall/ag/ag-overview

## Creation of the container instance with DVWA (without WAF)

This instance will be used without the WAF, so it can be access directly with its DNS_NAME_LABEL, which is `DVWA-27433.eastus.azurecontainer.io`

Each command is run independently

```bash
# The container should belong to a resource group (As well as all other resources)
az group create --name CSEC --location eastus 

# random number in the label for the domain
DNS_NAME_LABEL=DVWA-$RANDOM 

# create the container
az container create \
  --resource-group CSEC \
  --name dvwa \
  --image vulnerables/web-dvwa \
  --ports 80 \
  --dns-name-label $DNS_NAME_LABEL \
  --location eastus

# See the container details
az container show \
  --resource-group CSEC \
  --name dvwa

# Stop the container
az container stop \
  --resource-group CSEC \
  --name dvwa

# Start the container
az container start \
  --resource-group CSEC \
  --name dvwa
```

Note: we should wait a few minutes before starting the container again, otherwise the deployment won't be completed under de resource group and the container image will appear with a 'resource not found' message.


![alt text](Screenshot_20240501_102358.png)

# WAF
## What is a WAF?

The Azure Web Application Firewall (WAF) is a cloud-based service crafted to shield web applications from common cyber threats like SQL injection and cross-site scripting. It boasts swift deployment, providing a comprehensive view of the environment to effectively counter malicious attacks. Aligned with the Open Web Application Security Project (OWASP) top 10 security risks, Azure WAF ensures robust protection.

Its **main features** include:

* Custom and Managed Rule Sets: Users can create personalized rules or utilize preconfigured managed rule sets to effectively thwart malicious attacks.
* Real-time Visibility and Security Alerts: Users receive instant insight into their environment, enabling swift responses to potential threats. Monitoring attacks against web applications is facilitated through a real-time WAF log integrated with Azure Monitor.
* REST API Support: Full support for REST APIs allows for the automation of DevOps processes, streamlining security management.
* Operating within the Azure Application Gateway framework, Azure WAF functions as a layer 7 web traffic load balancer, enabling sophisticated routing decisions based on attributes like URI path and host headers.

## How does it work?

Utilizing the Core Rule Set (CRS) from OWASP, Azure WAF's underlying engine inspects incoming traffic to detect potential threats. Users can configure WAF policies and associate them with application gateways, ensuring granular protection for individual web applications.

**Key capabilities** of Azure WAF include:

*  Protection against common web attacks such as SQL injection, cross-site scripting, command injection, and HTTP protocol violations.
*  Configurable request size limits and exclusion lists for tailored security measures.
*  Creation of custom rules to address specific application requirements.
*  Geo-filtering capabilities to restrict access based on geographic location.
*  Bot mitigation ruleset to defend against automated bot attacks.
*  Inspection of JSON and XML in request bodies for comprehensive security coverage.

Azure WAF operates in **two modes**:

* Detection Mode: Monitors and logs threat alerts without blocking incoming requests, facilitating analysis and monitoring of potential security incidents.
* Prevention Mode: Actively blocks intrusions and attacks detected by the rules, issuing a "403 unauthorized access" exception to the attacker and recording incidents in WAF logs.

Users can configure actions for matched rule conditions, including allowing, blocking, logging, or assigning an anomaly score, providing tailored responses to diverse security threats.


## WAF setup
There are several resources we need to set up in order to deploy a web aplication (such as DVWA) with WAF enabled. Also all of these resources are under the same resource group we created earlier (CSEC):

![alt text](Screenshot_20240501_101614.png)


### waf-dvwa (Container instance)
  
As we don't want this container to be access directly, we need to set it up with a different configuration. 
![alt text](Screenshot_20240430_111142.png)

Mainly the network part with a private IP, for which we have to create a virtual network (CSEC-vnet)
![alt text](Screenshot_20240430_111230.png)

#### CSEC-vnet (Virtual network)
To create the vnet, we can click where it says new and a tab will open where we only need to name it and accept the defaults:

![alt text](image.png)

### waf-logs (Log analytics workspace)
We need to create a Log analytics workspace to store and query the logs from the 
Application gateway and WAF logs. To create it, we only need to name it and use the defaults:

![alt text](Screenshot_20240430_111530.png)

### waf-gateway (Application gateway)
Since a WAF (on azure) needs to be asociated with an Application gateway, which is a load balancer, we have to create it first with the following configuration:

![alt text](Screenshot_20240430_112923.png)
Note that we set up autoscaling but it is not really necesary for out testing purposes on DVWA. But it is important that we select tier `WAF V2`.
#### waf-policy (Application Gateway WAF policy)
For the WAF policy, we select create one and name it in the opened tab:
![alt text](image-1.png)
#### waf-vnet (Virtual network)
We also need to create another vnet for the Application Gateway (the steps are the same as for the CSEC-vnet). Note that currently the Application Gateway and the DVWA container are in separated vnet that we will connect later.


#### waf-public-ip (Public IP address)
In the Frontend configuration we need to create a `public` IP address that is the one we are going to connect to our application (Through the WAF policies).
To create it, we select create and only name it (using the defaults):
![alt text](Screenshot_20240430_112929.png)

#### waf-pool (Backend Pool)
Next, in backends, we need to create a backend pool without targets:

![alt text](image-2.png)

![alt text](Screenshot_20240430_112934.png)

Note that we will come back later to add our container instance as the target backend, that is where the Application Gateway will redirect the traffic to.

#### waf-route-rule (Routing rule)
Next, in configuration we need to create a routing rule as is the only missing part:
![alt text](image-3.png)

We will mainly use defaults for each section:
![alt text](image-4.png)
![alt text](Screenshot_20240501_114922.png)

In the backend targets we need to create backend setting with this configuration:
![alt text](Screenshot_20240501_114840.png)

#### Peering

Now, we need to allow traffic from the waf-vnet into the CSEC-vnet. To do so we create a peering under the waf-vnet resource page with the following configuration:
![alt text](Screenshot_20240430_113943.png)

## WAF testing and logging

Our application (DVWA) is now set up under WAF policies and can be access at `51.8.97.212`. However, as by default the WAF is set up in `detection` mode, we need to change that setting to `prevention` on its resource page:

![alt text](image-5.png)

As we mentioned earlier, detect mode will log possible attacks, but won't block them.

Lastly, the logs can be accesed and queried under the resource `waf-logs` that we created earlier (WAF logs are in the table `AGWFirewallLogs`):
![alt text](image-6.png)

# Attacking DVWA

We can perform attacks on the `dvwa` instance at `DVWA-27433.eastus.azurecontainer.io` or the `waf-dvwa` instance at `51.8.97.212`. However, on the last one we will only be able to perform them if the `waf-policy` is on `detection mode`, otherwise they are blocked automatically.

## Examples

We will perform a command injection and SQL injection attacks to see the WAF response and its logs:

### Command injection

We try to inject an `ls` command:
![alt text](Screenshot_20240501_121941.png)

It's blocked by the WAF and we're redirected to a 403 error page:
![alt text](Screenshot_20240501_122001.png)

In the logs we can see that severar rules checked the HTTP request headers and got a match for a command injection attack. As the 'Detailed data` column shows, `&&` and `&& ls` is a match. And finally, as the anomaly score was exceeded, it is blocked:

![alt text](Screenshot_20240501_122438.png)

### SQL injection
Similar to the command injection, we tried to inject SQL and was blocked by the WAF:
![alt text](Screenshot_20240501_122507.png)

![alt text](Screenshot_20240501_122510.png)

In this case there are 6 rules that matched our query as an injection attack, giving us a higher anomally score and thus blocking the request:
![alt text](Screenshot_20240501_122747.png)
# Red hat Developers Istio tutorial

There are three different and super simple microservices in this system and they are chained together in the following sequence:

customer -> preference -> recommendation

For now, they have a simple exception handling solution for dealing with
a missing dependent service, it just returns the error message to the end-user.

**Table of Contents**

<!-- toc -->

* [Installation](#installation)
* [Workshop](#workshop)
  * [Deployment and proxy injection](#deployment-and-proxy-injection)
    * [Deploy customer](#deploy-customer)
    * [Deploy preference](#deploy-preference)
    * [Deploy recommendation](#deploy-recommendation)
    * [Updating redeploying code](#updating-redeploying-code)
  * [Monitoring](#monitoring)
  * [Distributed Tracing](#distributed-tracing)
  * [Routes rules 101](#4---route-rules-101)
  * [Smart routing](#5---smart-routing)
  * [SLA and Error handling](#sla-and-error-handling)
    * [Fault injection](#fault-injection)
    * [Remediation](#remediation)
  * [Security](#security)
    * [Whitelist](#whitelist)
    * [Blacklist](#blacklist)
    * [Egress](#egress)
    * [Mutual TLS](#mutual-tls)
* [Tips & Tricks](#tips-tricks)

<!-- toc stop -->


## Installation

```bash
#!/bin/bash
curl -L https://github.com/istio/istio/releases/download/0.5.0/istio-0.5.0-osx.tar.gz | tar xz
cd istio-0.5.0
export ISTIO_HOME=$(pwd)
export PATH=$ISTIO_HOME/bin:$PATH
```

From `$ISTIO_HOME`:

```bash
oc login $(minishift ip):8443 -u admin -p admin
oc adm policy add-scc-to-user anyuid -z istio-ingress-service-account -n istio-system
oc adm policy add-scc-to-user anyuid -z default -n istio-system
oc create -f install/kubernetes/istio.yaml
oc project istio-system
oc expose svc istio-ingress
```

Wait for Istio's components to be ready

```bash
oc get pods
NAME                             READY     STATUS    RESTARTS   AGE
istio-ca-1363003450-tfnjp        1/1       Running   0          4m
istio-ingress-1005666339-vrjln   1/1       Running   0          4m
istio-mixer-465004155-zn78n      3/3       Running   0          5m
istio-pilot-1861292947-25hnm     2/2       Running   0          4m
```

And if you need quick access to the OpenShift console

```bash
minishift console
```

Note: on your first launch of the OpenShift console via minishift, you will like receive a warning with
"Your connection is not private", it depends on your browser type and settings.  Simply select "Proceed to 192.168.99.100 (unsafe)" to bypass the warning.

For minishift, with the admin-user addon, the user is "admin" and the password is "admin"

## Workshop

### 1 - Deployment and proxy injection

#### Setup environment

```bash
eval $(minishift oc-env)
eval $(minishift docker-env)
oc login $(minishift ip):8443 -u admin -p admin
```

#### Deploy customer

Make sure you have are logged in

```bash
oc whoami
```

and you have setup the project/namespace

```bash
oc new-project tutorial
oc adm policy add-scc-to-user privileged -z default -n tutorial
```

Then clone the git repository

```bash
git clone https://github.com/redhat-developer-demos/istio-tutorial
cd istio-tutorial
```

 Start deploying the microservice projects, starting with customer

```bash
cd customer

mvn clean package

docker build -t example/customer .

docker images | grep customer
```

Note: Your very first docker build will take a bit of time as it downloads all the layers.  Subsequent rebuilds of the docker image, updating only the jar/app layer will be very fast.

Now let's deploy the customer pod with its sidecar

```bash
oc apply -f <(istioctl kube-inject -f src/main/kubernetes/Deployment.yml) -n tutorial

oc create -f src/main/kubernetes/Service.yml -n tutorial
```

Since customer is the forward most microservice (customer -> preference -> recommendation), let's add an OpenShift Route that exposes that endpoint.

```bash
oc expose service customer

oc get route

oc get pods -w
```

Waiting for Ready 2/2, to break out of the waiting use "ctrl-c"

Then test the customer endpoint

```bash
curl customer-tutorial.$(minishift ip).nip.io
```

You should see the following error because preference and recommendation are not yet deployed.

```bash
customer => I/O error on GET request for "http://preference:8080": preference; nested exception is java.net.UnknownHostException: preference
```

Also review the logs

```bash
stern customer -c customer
```

You should see a stacktrace containing this cause:

```bash
org.springframework.web.client.ResourceAccessException: I/O error on GET request for "http://preference:8080": preference; nested exception is java.net.UnknownHostException: preference
```

Back to the main istio-tutorial directory

```bash
cd ..
```

#### Deploy preference

```bash
cd preference

mvn clean package

docker build -t example/preference .

docker images | grep preference

oc apply -f <(istioctl kube-inject -f src/main/kubernetes/Deployment.yml) -n tutorial

oc create -f src/main/kubernetes/Service.yml

oc get pods -w
```

Wait for the Ready 2/2

```bash
curl customer-tutorial.$(minishift ip).nip.io
```

It will respond with an error since recommendation is not yet deployed.
Note: We could make this a bit more resilent in a future iteration of this tutorial

```bash
customer => 503 preference => I/O error on GET request for "http://recommendation:8080": recommendation; nested exception is java.net.UnknownHostException: recommendation
```

and check out the logs

```bash
stern preference -c preference
```

You should see a stacktrace containing this cause:

```bash
org.springframework.web.client.ResourceAccessException: I/O error on GET request for "http://recommendation:8080": recommendation; nested exception is java.net.UnknownHostException: recommendation
```

Back to the main istio-tutorial directory

```bash
cd ..
```

#### Deploy recommendation

Note: The tag `v1` at the end of the image name is important.  We will be creating a v2 version of recommendation later in this tutorial. Having both a v1 and v2 version of the recommendation code will allow us to exercise some interesting aspects of Istio's capabilities.

```bash
cd recommendation

mvn clean package

docker build -t example/recommendation:v1 .

docker images | grep recommendation

oc apply -f <(istioctl kube-inject -f src/main/kubernetes/Deployment.yml) -n tutorial

oc create -f src/main/kubernetes/Service.yml

oc get pods -w

curl customer-tutorial.$(minishift ip).nip.io
```

it returns

```bash
customer => preference => recommendation v1 from '99634814-sf4cl': 1
```

and you can monitor the recommendation logs with

```bash
stern recommendation -c recommendation
```

Back to the main istio-tutorial directory

```bash
cd ..
```

#### Updating redeploying code

When you wish to change code (e.g. editing the .java files) and wish to "redeploy", simply:

```bash
cd {servicename}

vi src/main/java/com/redhat/developer/demos/{servicename}/{Servicename}{Controller|Verticle}.java
```

Make your edits and esc-w-q

```bash
mvn clean package
docker build -t example/{servicename} .

oc get pods -o jsonpath='{.items[*].metadata.name}' -l app={servicename}
oc get pods -o jsonpath='{.items[*].metadata.name}' -l app={servicename},version=v1

oc delete pod -l app={servicename},version=v1
```

Why the delete pod?

Based on the Deployment configuration, Kubernetes/OpenShift will recreate the pod, based on the new docker image as it attempts to keep the desired replicas available

```bash
oc describe deployment {servicename} | grep Replicas
```

### 2 - Monitoring

From `$ISTIO_HOME`:

```bash
oc adm policy add-scc-to-user anyuid -z grafana -n istio-system
oc adm policy add-scc-to-user anyuid -z prometheus -n istio-system
oc apply -f install/kubernetes/addons/prometheus.yaml
oc apply -f install/kubernetes/addons/grafana.yaml
oc expose svc grafana
oc expose svc prometheus
```

Send a bunch of request to the customer service:

```bash
for i in {1..100}; do curl "customer-tutorial.$(minishift ip).nip.io"; done
```

#### Istio dashboard

#### Mixer dashboard

Internal metrics on everything running into `istio-system` project.

#### Custom metrics

Istio also allows you to specify custom metrics which can be seen inside of the Prometheus dashboard

From the `istio-tutorial` directoy:

```bash
istioctl create -f istiofiles/recommendation_requestcount.yml -n istio-system
```

Send a bunch of request and check next request within Prometheus:

```bash
round(increase(istio_recommendation_request_count{destination="recommendation.tutorial.svc.cluster.local" }[60m]))
```

#### Custom log entries

From the `istio-tutorial` directoy:

```bash
istioctl create -f istiofiles/recommendation_entry.yml -n istio-system
```
Send a bunch of requests and connect to Mixer logs to filter new `recommendationentry` entries:

```bash
 oc logs istio-mixer-2464598866-rnhh8 -n istio-system -c mixer -f | grep recommendationentry
 ```

```bash
{"level":"warn","time":"2018-03-12T12:15:37.218825Z","instance":"recommendationentry.logentry.istio-system","destination":"recommendation","latency":"1.503ms","responseCode":200,"responseSize":48,"source":"preference","user":"unknown"}
```

Clean up

```bash
istioctl delete -f istiofiles/recommendation_requestcount.yml -n istio-system
istioctl delete -f istiofiles/recommendation_entry.yml -n istio-system
```

### 3 - Distributed tracing

Being already logged on `istio-system` project:

```bash
oc process -f https://raw.githubusercontent.com/jaegertracing/jaeger-openshift/master/all-in-one/jaeger-all-in-one-template.yml | oc create -f -
```

#### Explore Jaeger

![alt text](readme_images/jaegerUI.png "Jaeger with Customer")

#### Impacts on source code

Tracing requires a bit of work on the Java side. Each microservice needs to pass on the headers which are used to enable the traces.

https://github.com/jchraibi/istio-tutorial/blob/master/customer/src/main/java/com/redhat/developer/demos/customer/tracing/HttpHeaderForwarderHandlerInterceptor.java

and

https://github.com/jchraibi/istio-tutorial/blob/master/customer/src/main/java/com/redhat/developer/demos/customer/CustomerApplication.java#L21-L31

### 4 - Route rules 101

### Istio RouteRule Changes

#### recommendation:v2

We can experiment with Istio routing rules by making a change to RecommendationVerticle.java like the following and creating a "v2" docker image.

```java
    private static final String RESPONSE_STRING_FORMAT = "recommendation v2 from '%s': %d\n";
```

The "v2" tag during the docker build is significant.

There is also a 2nd deployment.yml file to label things correctly

```bash
cd recommendation

mvn clean package

docker build -t example/recommendation:v2 .

docker images | grep recommendation
example/recommendation                  v2                  c31e399a9628        5 seconds ago       438MB
example/recommendation                  v1              f072978d9cf6        8 minutes ago      438MB
```

*Important:* We have a 2nd Deployment to manage the v2 version of recommendation.  

```bash
oc apply -f <(istioctl kube-inject -f src/main/kubernetes/Deployment-v2.yml) -n tutorial

oc get pods -w
```

Wait for those pods to show "2/2", the istio-proxy/envoy sidecar is part of that pod

```bash
NAME                                  READY     STATUS    RESTARTS   AGE
customer-3600192384-fpljb             2/2       Running   0          17m
preference-243057078-8c5hz           2/2       Running   0          15m
recommendation-v1-60483540-9snd9     2/2       Running   0          12m
recommendation-v2-2815683430-vpx4p   2/2       Running   0         15s
```

and test the customer endpoint

```bash
curl customer-tutorial.$(minishift ip).nip.io
```

you likely see "customer => preference => recommendation v1 from '99634814-d2z2t': 3", where '99634814-d2z2t' is the pod running v1 and the 3 is basically the number of times you hit the endpoint.

```
curl customer-tutorial.$(minishift ip).nip.io
```

you likely see "customer => preference => recommendation v2 from '2819441432-5v22s': 1" as by default you get round-robin load-balancing when there is more than one Pod behind a Service

Send several requests to see their responses

```bash
#!/bin/bash
while true
do curl customer-tutorial.$(minishift ip).nip.io
sleep .1
done
```

The default Kubernetes/OpenShift behavior is to round-robin load-balance across all available pods behind a single Service.  Add another replica of recommendation-v2 Deployment.

```bash
oc scale --replicas=2 deployment/recommendation-v2
```

Now, you will see two requests into the v2 and one for v1.

```bash
customer => preference => recommendation v1 from '2819441432-qsp25': 29
customer => preference => recommendation v2 from '99634814-sf4cl': 37
customer => preference => recommendation v2 from '99634814-sf4cl': 38
```

Scale back to a single replica of the recommendation-v2 Deployment

```bash
oc scale --replicas=1 deployment/recommendation-v2
```

and back to the main directory

```bash
cd ..
```

### Changing Istio RouteRules

#### All users to recommendation:v2

From the main istio-tutorial directory,

```bash
istioctl create -f istiofiles/route-rule-recommendation-v2.yml -n tutorial

curl customer-tutorial.$(minishift ip).nip.io
```

you should only see v2 being returned

#### All users to recommendation:v1

Note: "replace" instead of "create" since we are overlaying the previous rule

```bash
istioctl replace -f istiofiles/route-rule-recommendation-v1.yml -n tutorial

istioctl get routerules -n tutorial

istioctl get routerule recommendation-default -o yaml -n tutorial
```

#### All users to recommendation v1 and v2

By simply removing the rule

```bash
istioctl delete routerule recommendation-default -n tutorial
```

and you should see the default behavior of load-balancing between v1 and v2

```bash
curl customer-tutorial.$(minishift ip).nip.io
```

#### Split traffic between v1 and v2

Canary Deployment scenario: push v2 into the cluster but slowly send end-user traffic to it, if you continue to see success, continue shifting more traffic over time

```bash
oc get pods -l app=recommendation -n tutorial
NAME                                  READY     STATUS    RESTARTS   AGE
recommendation-v1-3719512284-7mlzw   2/2       Running   6          2h
recommendation-v2-2815683430-vn77w   2/2       Running   0          1h
```

Create the routerule that will send 90% of requests to v1 and 10% to v2

```bash
istioctl create -f istiofiles/route-rule-recommendation-v1_and_v2.yml -n tutorial
```

and send in several requests

```bash
#!/bin/bash
while true
do curl customer-tutorial.$(minishift ip).nip.io
sleep .1
done
```

In another terminal, change the mixture to be 75/25

```bash
istioctl replace -f istiofiles/route-rule-recommendation-v1_and_v2_75_25.yml -n tutorial
```

Clean up

```bash
istioctl delete routerule recommendation-v1-v2 -n tutorial
```

### Defining Rules precedence
The `precedence` of the routing rules matters. The higher the value of precedence, that rule takes higher precedence.
Let's look at these 2 route rules:

```
apiVersion: config.istio.io/v1alpha2
kind: RouteRule
metadata:
  name: routerule_v1
spec:
  destination:
    name: some_service_v1
  precedence: 2
  route:
  - labels:
      version: v1
```

```
apiVersion: config.istio.io/v1alpha2
kind: RouteRule
metadata:
  name: routerule_v2
spec:
  destination:
    name: some_service_v2
  precedence: 3
  route:
  - labels:
      version: v2
```
In this case, if both route rules are applied, the traffic will always be redirected to v2 because it has a precedence of 3 which is higher than precendence:2 of routerule_v1.


### 5 - Smart Routing
### Smart routing based on user-agent header (Canary Deployment)

What is your user-agent?

https://www.whoishostingthis.com/tools/user-agent/

Note: the "user-agent" header being forwarded in the Customer and Preferences controllers in order for route rule modications around recommendation

#### Set recommendation to all v1

```bash
istioctl create -f istiofiles/route-rule-recommendation-v1.yml -n tutorial
```

#### Set Safari users to v2

```bash
istioctl create -f istiofiles/route-rule-safari-recommendation-v2.yml -n tutorial

istioctl get routerules -n tutorial
```

and test with a Safari (or even Chrome on Mac since it includes Safari in the string).  Safari only sees v2 responses from recommendation

and test with a Firefox browser, it should only see v1 responses from recommendation.

There are two ways to get the URL for your browser:

```bash
minishift openshift service customer --in-browser
```

That will open the openshift service `customer` in browser

Or

if you need just the url alone:

```bash
minishift openshift service customer --url
http://customer-tutorial.192.168.99.102.nip.io
```

You can also attempt to use the curl -A command to test with different user-agent strings.  

```bash
curl -A Safari customer-tutorial.$(minishift ip).nip.io
curl -A Firefox customer-tutorial.$(minishift ip).nip.io
```

You can describe the routerule to see its configuration

```bash
oc describe routerule recommendation-safari -n tutorial
```

Remove the Safari rule

```bash
istioctl delete routerule recommendation-safari -n tutorial
```

#### Set mobile users to v2

```bash
istioctl create -f istiofiles/route-rule-mobile-recommendation-v2.yml -n tutorial

curl -A "Mozilla/5.0 (iPhone; U; CPU iPhone OS 4(KHTML, like Gecko) Version/5.0.2 Mobile/8J2 Safari/6533.18.5" curl -A Safari customer-tutorial.$(minishift ip).nip.io
```

#### Clean up

```bash
istioctl delete routerule recommendation-mobile -n tutorial
```

### Mirroring Traffic (Dark Launch)

```bash
oc get pods -l app=recommendation -n tutorial
```

You should have 2 pods for recommendation based on the steps above

```bash
istioctl get routerules -n tutorial
```

You should have NO routerules
if so "istioctl delete routerule rulename -n tutorial"

Make sure you are in the main directory of "istio-tutorial"

```bash
istioctl create -f istiofiles/route-rule-recommendation-v1-mirror-v2.yml -n tutorial

curl customer-tutorial.$(minishift ip).nip.io
```

Check the logs of recommendation-v2

```bash
oc logs -f `oc get pods|grep recommendation-v2|awk '{ print $1 }'` -c recommendation
```


#### Clean up

```bash
istioctl delete routerule recommendation-mirror -n tutorial
```



### Load Balancer

By default, you will see "round-robin" style load-balancing, but you can change it up, with the RANDOM option being fairly visible to the naked eye.

Add another v2 pod to the mix

```bash
oc scale deployment recommendation-v2 --replicas=2 -n tutorial
```

Wait a bit (oc get pods -w to watch)
and curl the customer endpoint many times

```bash
curl customer-tutorial.$(minishift ip).nip.io
```

Add a 3rd v2 pod to the mix

```bash
oc scale deployment recommendation-v2 --replicas=3 -n tutorial

oc get pods -n tutorial
NAME                                  READY     STATUS    RESTARTS   AGE
customer-1755156816-cjd2z             2/2       Running   0          1h
preference-3336288630-2cc6f          2/2       Running   0          1h
recommendation-v1-3719512284-bn42p   2/2       Running   0          59m
recommendation-v2-2815683430-97nnf   2/2       Running   0          43m
recommendation-v2-2815683430-d49n6   2/2       Running   0          51m
recommendation-v2-2815683430-tptf2   2/2       Running   0          33m
```

Wait for those 2/2 (two containers in each pod) and then poll the customer endpoint

```bash
#!/bin/bash
while true
do curl customer-tutorial.$(minishift ip).nip.io
sleep .1
done
```

The results should follow a fairly normal round-robin distribution pattern

```bash
customer => preference => recommendation v1 from '99634814-d2z2t': 1145
customer => preference => recommendation v2 from '2819441432-525lh': 1
customer => preference => recommendation v2 from '2819441432-rg45q': 2
customer => preference => recommendation v2 from '2819441432-bs5ck': 181
customer => preference => recommendation v1 from '99634814-d2z2t': 1146
customer => preference => recommendation v2 from '2819441432-rg45q': 3
customer => preference => recommendation v2 from '2819441432-rg45q': 4
customer => preference => recommendation v2 from '2819441432-bs5ck': 182
```

Now, add the Random LB DestinationPolicy

```bash
istioctl create -f istiofiles/recommendation_lb_policy_app.yml -n tutorial
```

And you should see a different pattern of which pod is being selected

```bash
customer => preference => recommendation v2 from '2819441432-rg45q': 10
customer => preference => recommendation v2 from '2819441432-525lh': 3
customer => preference => recommendation v2 from '2819441432-rg45q': 11
customer => preference => recommendation v1 from '99634814-d2z2t': 1153
customer => preference => recommendation v1 from '99634814-d2z2t': 1154
customer => preference => recommendation v1 from '99634814-d2z2t': 1155
customer => preference => recommendation v2 from '2819441432-rg45q': 12
customer => preference => recommendation v2 from '2819441432-525lh': 4
customer => preference => recommendation v2 from '2819441432-525lh': 5
customer => preference => recommendation v2 from '2819441432-rg45q': 13
customer => preference => recommendation v2 from '2819441432-rg45q': 14
```

Clean up

```bash
istioctl delete -f istiofiles/recommendation_lb_policy_app.yml -n tutorial

oc scale deployment recommendation-v2 --replicas=1 -n tutorial
```

### Rate Limiting

**Note**: currently not working

Here we will limit the number of concurrent requests into recommendation v2

Now apply the rate limit handler

```bash
istioctl create -f istiofiles/recommendation_rate_limit_handler.yml
```

Now setup the requestcount quota

```bash
istioctl create -f istiofiles/rate_limit_rule.yml
```

Throw some requests at customer

```bash
#!/bin/bash
while true
do curl customer-tutorial.$(minishift ip).nip.io
sleep .1
done
```

You should see some 429 errors:

```bash
customer => preference => recommendation v2 from '2819441432-f4ls5': 108
customer => preference => recommendation v1 from '99634814-d2z2t': 1932
customer => preference => recommendation v2 from '2819441432-f4ls5': 109
customer => preference => recommendation v1 from '99634814-d2z2t': 1933
customer => 503 preference => 429 Too Many Requests
customer => preference => recommendation v1 from '99634814-d2z2t': 1934
customer => preference => recommendation v2 from '2819441432-f4ls5': 110
customer => preference => recommendation v1 from '99634814-d2z2t': 1935
customer => 503 preference => 429 Too Many Requests
customer => preference => recommendation v1 from '99634814-d2z2t': 1936
customer => preference => recommendation v2 from '2819441432-f4ls5': 111
customer => preference => recommendation v1 from '99634814-d2z2t': 1937
customer => 503 preference => 429 Too Many Requests
customer => preference => recommendation v1 from '99634814-d2z2t': 1938
customer => preference => recommendation v2 from '2819441432-f4ls5': 112
```

Clean up

```bash
istioctl delete -f istiofiles/rate_limit_rule.yml

istioctl delete -f istiofiles/recommendation_rate_limit_handler.yml
```

### 6 - SLA and Error handling

#### Chaos engineering

Apply some chaos engineering by throwing in some HTTP errors or network delays. Understanding failure scenarios is a critical aspect of microservices architecture (aka distributed computing)

##### Fault injection

By default, recommendation v1 and v2 are being randomly load-balanced as that is the default behavior in Kubernetes/OpenShift

You can inject 503's, for approximately 50% of the requests.

From the `istio-tutorial` directory:

```bash
istioctl create -f istiofiles/route-rule-recommendation-503.yml -n tutorial
```

with the following result:

```bash
customer => preference => recommendation v1 from '2793872006-rq7fs': 1271
customer => 503 preference => 503 fault filter abort
customer => preference => recommendation v2 from '3406724218-kdftb': 11
```

You can visualize some error spans into Jaeger and the effect on metrics on the Graphana Istio Dashboard.

Clean up

```bash
istioctl delete routerule recommendation-503 -n tutorial
```

##### Delay injection

The most insidious of possible distributed computing faults is not a "down" service but a service that is responding slowly, potentially causing a cascading failure in your network of services.

```bash
istioctl create -f istiofiles/route-rule-recommendation-delay.yml -n tutorial
```

And hit the customer endpoint many times.

You will notice many requests to the customer endpoint now have a delay.
If you are monitoring the logs for recommendation v1 and v2, you will also see the delay happens BEFORE the recommendation service is actually called

As an exercise, change the `recommendation-delay` route rule configuration on the fly, lowering percent of requests and delay injected.

Clean up

```bash
istioctl delete routerule recommendation-delay -n tutorial
```

#### Remediation

##### Retry

Instead of failing immediately, retry the Service N more times

We will use Istio and return 503's about 50% of the time. Send all users to v2 which will throw out some 503's

```bash
istioctl create -f istiofiles/route-rule-recommendation-v2_503.yml -n tutorial
```

Now, if you hit the customer endpoint several times, you should see some 503's

```bash
customer => preference => recommendation v2 from '3406724218-kdftb': 458
customer => 503 preference => 503 fault filter abort
customer => 503 preference => 503 fault filter abort
customer => 503 preference => 503 fault filter abort
customer => 503 preference => 503 fault filter abort
customer => preference => recommendation v2 from '3406724218-kdftb': 459
customer => 503 preference => 503 fault filter abort
customer => 503 preference => 503 fault filter abort
customer => preference => recommendation v2 from '3406724218-kdftb': 460
```

Now add the retry rule

```bash
istioctl create -f istiofiles/route-rule-recommendation-v2_retry.yml -n tutorial
```

and after a few seconds, things will settle down and you will see it work every time

```bash
customer => preference => recommendation v2 from '3406724218-kdftb': 641
customer => preference => recommendation v2 from '3406724218-kdftb': 642
customer => preference => recommendation v2 from '3406724218-kdftb': 643
customer => preference => recommendation v2 from '3406724218-kdftb': 644
customer => preference => recommendation v2 from '3406724218-kdftb': 645
customer => preference => recommendation v2 from '3406724218-kdftb': 646
customer => preference => recommendation v2 from '3406724218-kdftb': 647
customer => preference => recommendation v2 from '3406724218-kdftb': 648
```

You can see the active RouteRules via:

```bash
$ istioctl get routerules -n tutorial
NAME			KIND					NAMESPACE
recommendation-v2-503	RouteRule.v1alpha2.config.istio.io	tutorial
recommendation-v2-retry	RouteRule.v1alpha2.config.istio.io	tutorial
```

or via:

```bash
$ oc get routerules -n tutorial
NAME                      AGE
recommendation-v2-503     41m
recommendation-v2-retry   41m
```

Delete route `recommendation-v2-retry`, check the previously avoided behaviour. Then remove `recommandation-v2-503` rule and fall-back to default:50/50 load-balancing between v1 and v2.

##### Timeout

Wait only N seconds before giving up and failing.  At this point, no other route rules should be in effect.  `oc get routerules` and `oc delete routerule <rulename>` if there are some.

First, introduce some wait time in `recommendation v2` by uncommenting the line that calls the `timeout()` method. Update `RecommendationVerticle.java` making it a slow performer with a 3 second delay.

```java
    @Override
    public void start() throws Exception {
        Router router = Router.router(vertx);
        router.get("/").handler(this::logging);
        router.get("/").handler(this::timeout);
        router.get("/").handler(this::getRecommendations);
        router.get("/misbehave").handler(this::misbehave);
        router.get("/behave").handler(this::behave);

        HealthCheckHandler hc = HealthCheckHandler.create(vertx);
        hc.register("dummy-health-check", future -> future.complete(Status.OK()));
        router.get("/health").handler(hc);

        vertx.createHttpServer().requestHandler(router::accept).listen(8080);
    }
```

Rebuild and redeploy.

```bash
cd recommendation
mvn clean package
docker build -t example/recommendation:v2 .
oc delete pod -l app=recommendation,version=v2 -n tutorial
cd ..
```

Hit the customer endpoint a few times, to see the load-balancing between v1 and v2 but with v2 taking a bit of time to respond (3 seconds as per the `timeout()` handler added into `RecommendationVerticle.java` ;-))

Then add the timeout rule

```bash
istioctl create -f istiofiles/route-rule-recommendation-timeout.yml -n tutorial
```

You will see it return v1 OR `upstream request timeout` after waiting about 1 second

```bash
customer => preference => recommendation v1 from '2793872006-rq7fs': 1518
customer => 503 preference => 504 upstream request timeout
customer => preference => recommendation v1 from '2793872006-rq7fs': 1519
customer => 503 preference => 504 upstream request timeout
customer => preference => recommendation v1 from '2793872006-rq7fs': 1520
customer => 503 preference => 504 upstream request timeout
```

Clean up, delete the timeout rule

```bash
istioctl delete -f istiofiles/route-rule-recommendation-timeout.yml -n tutorial
```

### 7 - Circuit Breaker (LBR)

#### Fail Fast

First, make sure to uncomment `router.get("/").handler(this::timeout);` in the RecommendationVerticle.java:

```java
    Router router = Router.router(vertx);
    router.get("/").handler(this::logging);
    router.get("/").handler(this::timeout);
    router.get("/").handler(this::getRecommendations);
    router.get("/misbehave").handler(this::misbehave);
    router.get("/behave").handler(this::behave);
```

And follow the Updating & redeploying code steps to get this slower v2 deployed.

Second, you need to insure you have a `routerule` in place. Let's use a 50/50 split of traffic:

```bash
istioctl create -f istiofiles/route-rule-recommendation-v1_and_v2_50_50.yml -n tutorial
```
**Load test without circuit breaker**

Let's have a load-test using the `siege` command line tool. We'll have 20 clients sending 2 concurrent requests each:

```bash
siege -r 2 -c 20 -v customer-tutorial.$(minishift ip).nip.io
```

You should see an output similar to this one:

```bash
** SIEGE 4.0.4
** Preparing 20 concurrent users for battle.
The server is now under siege...
HTTP/1.1 200     0.11 secs:      74 bytes ==> GET  /
HTTP/1.1 200     0.11 secs:      74 bytes ==> GET  /
HTTP/1.1 200     0.14 secs:      74 bytes ==> GET  /
...
Transactions:		            40 hits
Availability:		        100.00 %
Elapsed time:		          6.24 secs
Data transferred:	        0.00 MB
Response time:		        1.66 secs
Transaction rate:	        6.41 trans/sec
Throughput:		            0.00 MB/sec
Concurrency:		         10.67
Successful transactions:    40
Failed transactions:	       0
Longest transaction:	    3.21
Shortest transaction:	    0.01
```
All of the requests to our system were successful, but it took some time to run the test, as the `v2` instance/pod was a slow performer.

But suppose that in a production system this 3s delay was caused by too many concurrent requests to the same instance/pod. We don't want multiple requests getting queued or making the instance/pod even slower. So we'll add a circuit breaker that will open whenever we have more than 1 request being handled by any instance/pod.

```bash
istioctl create -f istiofiles/recommendation_cb_policy_version_v2.yml -n tutorial

istioctl get destinationpolicies -n tutorial
```
**Load test with circuit breaker**

Let's see the behaviour when running `siege` again:

```bash
siege -r 2 -c 20 -v customer-tutorial.$(minishift ip).nip.io
```

```bash
** SIEGE 4.0.4
** Preparing 20 concurrent users for battle.
The server is now under siege...
HTTP/1.1 200     0.07 secs:      74 bytes ==> GET  /
HTTP/1.1 200     0.09 secs:      74 bytes ==> GET  /
HTTP/1.1 200     0.15 secs:      74 bytes ==> GET  /
HTTP/1.1 503     0.15 secs:      92 bytes ==> GET  /
HTTP/1.1 200     0.15 secs:      74 bytes ==> GET  /
HTTP/1.1 503     0.15 secs:      92 bytes ==> GET  /
HTTP/1.1 503     0.16 secs:      92 bytes ==> GET  /
...
Transactions:		          22 hits
Availability:		       55.00 %
Elapsed time:		        9.11 secs
Data transferred:	      0.00 MB
Response time:		      1.09 secs
Transaction rate:	      2.41 trans/sec
Throughput:		          0.00 MB/sec
Concurrency:		        2.63
Successful transactions:        22
Failed transactions:	          18
Longest transaction:	        6.10
Shortest transaction:	        0.01
```

You'll get a little more of 50 percent success in requests. That's the circuit breaker being opened whenever Istio detects more than 1 pending request being handled by the instance/pod tagged `v2`.

More information on the fields for the simple circuit-breaker
https://istio.io/docs/reference/config/istio.routing.v1alpha1.html#CircuitBreaker.SimpleCircuitBreakerPolicy

Clean up

```bash
istioctl delete routerule recommendation-v1-v2 -n tutorial
istioctl delete -f istiofiles/recommendation_cb_policy_version_v2.yml -n tutorial
```

#### Pool Ejection

Pool ejection or *outlier detection* is a resilience strategy that takes place whenever we have a pool of instances/pods to serve a client request. If the request is forwarded to a certain instance and it fails (e.g. returns a 50x error code), then Istio will eject this instance from the pool for a certain *sleep window*. In our example the sleep window is configured to be 15s. This increases the overall availability by making sure that only healthy pods participate in the pool of instances.

First, you need to insure you have a `routerule` in place. Let's use a 50/50 split of traffic:

```bash
istioctl create -f istiofiles/route-rule-recommendation-v1_and_v2_50_50.yml -n tutorial
```

You'll also need to scale the number of instances of `v2` deployment:

```bash
oc scale deployment recommendation-v2 --replicas=2 -n tutorial
```

**Test behavior without failing instances**

Throw some requests to the customer endpoint like this `for i in {1..10}; do curl "customer-tutorial.$(minishift ip).nip.io"; done`

You will see the load balancing 50/50 between the two different versions of the recommendation service. And within version v2, you will also see that some requests are handled by one pod and some requests are handled by the other pod.

**Test behavior with failing instance and without pool ejection**

Let's get the name for the pods of `recommendation-v2`:

```bash
oc get pods -l app=recommendation,version=v2
```

You should get something like:

```bash
NAME                                 READY     STATUS    RESTARTS   AGE
recommendation-v2-3406724218-g6n7b   2/2       Running   0          1h
recommendation-v2-3406724218-zv6dt   2/2       Running   0          37m
```

Now we'll get into one the pods and add some erratic behavior on it. Get one of the pod names from your system and replace on the following command accordingly:

```bash
oc rsh recommendation-v2-3406724218-zv6dt
```

You will be inside the application container of your pod `recommendation-v2-3406724218-zv6dt`. Now execute:

```bash
curl localhost:8080/misbehave
exit
```

Now throw some requests at the customer endpoint. You should see something like:

```bash
customer => 503 preference => 503 recommendation misbehavior from '3406724218-zv6dt'
customer => preference => recommendation v1 from '2793872006-rq7fs': 1599
customer => preference => recommendation v2 from '3406724218-g6n7b': 59
customer => preference => recommendation v1 from '2793872006-rq7fs': 1600
customer => 503 preference => 503 recommendation misbehavior from '3406724218-zv6dt'
customer => preference => recommendation v1 from '2793872006-rq7fs': 1601
customer => preference => recommendation v2 from '3406724218-g6n7b': 60
customer => 503 preference => 503 recommendation misbehavior from '3406724218-zv6dt'
customer => preference => recommendation v1 from '2793872006-rq7fs': 1602
```

**Test behavior with failing instance and with pool ejection**

Now let's add the pool ejection behavior:

```bash
istioctl create -f istiofiles/recommendation_cb_policy_pool_ejection.yml -n tutorial
```

2 identical `DestinationPolicy` objects are created and apply respectively to the `v1` and the `v2` pods/instances of `recommendation`. Throw some requesrs at the customer endpoint and you will see that whenever you get a failing request with 503 from the pod recommendation-v2-3406724218-zv6dt, it gets ejected from the pool, and it doesn't receive any more requests until the sleep window expires - which takes at least 15s.

```bash
customer => 503 preference => 503 recommendation misbehavior from '3406724218-zv6dt'
customer => preference => recommendation v2 from '3406724218-g6n7b': 68
customer => preference => recommendation v2 from '3406724218-g6n7b': 69
customer => preference => recommendation v1 from '2793872006-rq7fs': 1606
customer => preference => recommendation v2 from '3406724218-g6n7b': 70
customer => preference => recommendation v1 from '2793872006-rq7fs': 1607
```

Clean up

```bash
istioctl delete -f istiofiles/recommendation_cb_policy_pool_ejection.yml -n tutorial
```

### 8 - Ultimate resilience (JCH)

### 9 - Security

#### Access Control

##### Whitelist

We'll create a whitelist on the `preference` service to only allow requests from the `recommendation` service, which will make the `preference` service invisible to the `customer` service. Requests from the customer service to the preference service will return a 404 Not Found HTTP error code.

```bash
istioctl create -f istiofiles/acl-whitelist.yml -n tutorial
```

```bash
curl customer-tutorial.$(minishift ip).nip.io
customer => 404 NOT_FOUND:preferencewhitelist.listchecker.tutorial:customer is not whitelisted
```

Clean up

```bash
istioctl delete -f istiofiles/acl-whitelist.yml -n tutorial
```

##### Blacklist

We'll create a blacklist making the customer service blacklist to the preference service. Requests from the customer service to the preference service will return a 403 Forbidden HTTP error code.

```bash
istioctl create -f istiofiles/acl-blacklist.yml -n tutorial
```

```bash
curl customer-tutorial.$(minishift ip).nip.io
customer => 403 PERMISSION_DENIED:denycustomerhandler.denier.tutorial:Not allowed
```

Clean up

```bash
istioctl delete -f istiofiles/acl-blacklist.yml -n tutorial
```

As a complementary exercice, you may consider writing a `Denier` that does not allow accessing to `recommendation-v2` from `preference` service.

#### Egress

There are two examples of egress routing, one for httpbin.org and one for github. Egress routes allow you to apply rules to how internal services interact with external APIs/services.

Create a namespace/project to hold these egress examples

```bash
oc new-project istioegress
oc adm policy add-scc-to-user privileged -z default -n istioegress
```

##### HTTPBin Java App

Build and test the Egress application for HTTPBin.

```bash
cd egress/egresshttpbin/
mvn spring-boot:run
```

and from another terminal on your laptop, check the normal output of it:

```bash
curl localhost:8080
```

Now create the docker image and deploy it onto your minishift VM:

```bash
mvn clean package
docker build -t example/egresshttpbin:v1 .
docker run -it -p 8080:8080 --rm example/egresshttpbin:v1
```

and from another terminal on your laptop, check the normal output of it:

```bash
curl $(minishift ip):8080
```

Now deploy it on normal OpenShift project, exposing a route for it:

```bash
oc apply -f <(istioctl kube-inject -f src/main/kubernetes/Deployment.yml) -n istioegress
oc create -f src/main/kubernetes/Service.yml
oc expose service egresshttpbin
```

and check that it does not work anymore... yet.

```bash
curl egresshttpbin-istioegress.$(minishift ip).nip.io
```

Back to the main `istio-tutorial` directory:

##### GitHub Java App

Build and deploy on your minishitf VM the GitHub java app:

```bash
cd egress/egressgithub/
mvn clean package
docker build -t example/egressgithub:v1 .
docker run -it -p 8080:8080 --rm example/egressgithub:v1
```

and check that is does not work on another terminal:

```bash
$ curl $(minishift ip):8080
I/O error on GET request for "http://api.github.com:443/users": Unexpected end of file from server; nested exception is java.net.SocketException: Unexpected end of file from server
```

but it will work once istio-ized ;-) Deploy it as regular Deployment on OpenShift:

```bash
oc apply -f <(istioctl kube-inject -f src/main/kubernetes/Deployment.yml) -n istioegress
oc create -f src/main/kubernetes/Service.yml
oc expose service egressgithub
```

and check that it still does not work.

```bash
curl egressgithub-istioegress.$(minishift ip).nip.io
```

##### Istio-ize Egress

Unlike raw Kubernetes, Istio/Envoy block all outbound traffic from a pod (envoy intercepts all in/out traffic) as that is considered to be better/smarter for security reasons. One of the more interesting attack vectors (if you are a blackhat hacker) is to install a small process on a remote server, have it collect interesting data and "phone home". So, Istio/Envoy block that outbound traffic (pod to outside the service mesh). Therefore, you must explicitly open up the egress routes, kind of like opening up ports in the firewall.

```bash
istioctl create -f istiofiles/egress_httpbin.yml -n istioegress
```
Now we can test again our `egresshttpbin` service that way `curl egresshttpbin-istioegress.$(minishift ip).nip.io` and see complete response:

```bash
{
  "headers": {
    "Accept": "text/plain, application/json, application/*+json, */*",
    "Accept-Charset": "big5, big5-hkscs, cesu-8, euc-jp, euc-kr, gb18030, gb2312, gbk, ibm-thai, ibm00858, ibm01140, ibm01141, ibm01142, ibm01143, ibm01144, ibm01145, ibm01146, ibm01147, ibm01148, ibm01149, ibm037, ibm1026, ibm1047, ibm273, ibm277, ibm278, ibm280, ibm284, ibm285, ibm290, ibm297, ibm420, ibm424, ibm437, ibm500, ibm775, ibm850, ibm852, ibm855, ibm857, ibm860, ibm861, ibm862, ibm863, ibm864, ibm865, ibm866, ibm868, ibm869, ibm870, ibm871, ibm918, iso-2022-cn, iso-2022-jp, iso-2022-jp-2, iso-2022-kr, iso-8859-1, iso-8859-13, iso-8859-15, iso-8859-2, iso-8859-3, iso-8859-4, iso-8859-5, iso-8859-6, iso-8859-7, iso-8859-8, iso-8859-9, jis_x0201, jis_x0212-1990, koi8-r, koi8-u, shift_jis, tis-620, us-ascii, utf-16, utf-16be, utf-16le, utf-32, utf-32be, utf-32le, utf-8, windows-1250, windows-1251, windows-1252, windows-1253, windows-1254, windows-1255, windows-1256, windows-1257, windows-1258, windows-31j, x-big5-hkscs-2001, x-big5-solaris, x-compound_text, x-euc-jp-linux, x-euc-tw, x-eucjp-open, x-ibm1006, x-ibm1025, x-ibm1046, x-ibm1097, x-ibm1098, x-ibm1112, x-ibm1122, x-ibm1123, x-ibm1124, x-ibm1166, x-ibm1364, x-ibm1381, x-ibm1383, x-ibm300, x-ibm33722, x-ibm737, x-ibm833, x-ibm834, x-ibm856, x-ibm874, x-ibm875, x-ibm921, x-ibm922, x-ibm930, x-ibm933, x-ibm935, x-ibm937, x-ibm939, x-ibm942, x-ibm942c, x-ibm943, x-ibm943c, x-ibm948, x-ibm949, x-ibm949c, x-ibm950, x-ibm964, x-ibm970, x-iscii91, x-iso-2022-cn-cns, x-iso-2022-cn-gb, x-iso-8859-11, x-jis0208, x-jisautodetect, x-johab, x-macarabic, x-maccentraleurope, x-maccroatian, x-maccyrillic, x-macdingbat, x-macgreek, x-machebrew, x-maciceland, x-macroman, x-macromania, x-macsymbol, x-macthai, x-macturkish, x-macukraine, x-ms932_0213, x-ms950-hkscs, x-ms950-hkscs-xp, x-mswin-936, x-pck, x-sjis_0213, x-utf-16le-bom, x-utf-32be-bom, x-utf-32le-bom, x-windows-50220, x-windows-50221, x-windows-874, x-windows-949, x-windows-950, x-windows-iso2022jp",
    "Connection": "close",
    "Content-Type": "text/plain;charset=ISO-8859-1",
    "Host": "httpbin.org",
    "User-Agent": "Java/1.8.0_151",
    "X-B3-Sampled": "1",
    "X-B3-Spanid": "0a03424a24e4b6ce",
    "X-B3-Traceid": "0a03424a24e4b6ce",
    "X-Envoy-Decorator-Operation": "default-route",
    "X-Istio-Attributes": "Ch8KCXNvdXJjZS5pcBISMhAAAAAAAAAAAAAA//+sEQAOCkoKCnNvdXJjZS51aWQSPBI6a3ViZXJuZXRlczovL2VncmVzc2h0dHBiaW4tdjEtMzg1ODgxNTM0My1jYnJsbC5pc3Rpb2VncmVzcw==",
    "X-Ot-Span-Context": "0a03424a24e4b6ce;0a03424a24e4b6ce;0000000000000000"
  }
}
```

By the way, we can see now response contains OpenTracing headers and we can check trace is visible into Jaeger.

Now, apply the `egressrule` for github and execute the Java code that hits `api.github.com/users`

```bash
istioctl create -f istiofiles/egress_github.yml -n istioegress
```

Check everything's fine with `curl egressgithub-istioegress.$(minishift ip).nip.io`

Clean up

```bash
istioctl delete egressrule httpbin-egress-rule github-egress-rule -n istioegress
```


#### Mutual TLS

```bash
oc get configmap istio -o yaml -n istio-system | grep authPolicy | head -1
```

Uncomment it

Scale-up / scale down : istio-pilot, istio-ingress, istio-mixer

Scale-up / scale down : preference


## Tips & Tricks

Some tips and tricks that you might find handy

You have two containers in a pod

```bash
oc get pods -o jsonpath="{.items[*].spec.containers[*].name}" -l app=customer -n tutorial
```

From these images

```bash
oc get pods -o jsonpath="{.items[*].spec.containers[*].image}" -l app=customer -n tutorial
```

Get the pod ids

```bash
CPOD=$(oc get pods -o jsonpath='{.items[*].metadata.name}' -l app=customer -n tutorial)
PPOD=$(oc get pods -o jsonpath='{.items[*].metadata.name}' -l app=preference -n tutorial)
RPOD1=$(oc get pods -o jsonpath='{.items[*].metadata.name}' -l app=recommendation,version=v1 -n tutorial)
RPOD2=$(oc get pods -o jsonpath='{.items[*].metadata.name}' -l app=recommendation,version=v2 -n tutorial)
```

The pods all see each other's services

```bash
oc exec $CPOD -c customer -n tutorial curl http://preference:8080
oc exec $CPOD -c customer -n tutorial curl http://recommendation:8080
oc exec $RPOD2 -c recommendation -n tutorial curl http://customer:8080
```

```bash
oc exec $CPOD -c customer -n tutorial curl http://localhost:15000/routes > afile.json
```

Look for "route_config_name": "8080", you should see 3 entries for customer, preference and recommendation

```json
{
	"name": "8080",
	"virtual_hosts": [{
		"name": "customer.springistio.svc.cluster.local|http",
		"domains": ["customer:8080", "customer", "customer.springistio:8080", "customer.springistio", "customer.springistio.svc:8080", "customer.springistio.svc", "customer.springistio.svc.cluster:8080", "customer.springistio.svc.cluster", "customer.springistio.svc.cluster.local:8080", "customer.springistio.svc.cluster.local", "172.30.176.159:8080", "172.30.176.159"],
		"routes": [{
			"match": {
				"prefix": "/"
			},
			"route": {
				"cluster": "out.customer.springistio.svc.cluster.local|http",
				"timeout": "0s"
			},
			"decorator": {
				"operation": "default-route"
			}
		}]
	}, {
		"name": "preference.springistio.svc.cluster.local|http",
		"domains": ["preference:8080", "preference", "preference.springistio:8080", "preference.springistio", "preference.springistio.svc:8080", "preference.springistio.svc", "preference.springistio.svc.cluster:8080", "preference.springistio.svc.cluster", "preference.springistio.svc.cluster.local:8080", "preference.springistio.svc.cluster.local", "172.30.249.133:8080", "172.30.249.133"],
		"routes": [{
			"match": {
				"prefix": "/"
			},
			"route": {
				"cluster": "out.preference.springistio.svc.cluster.local|http",
				"timeout": "0s"
			},
			"decorator": {
				"operation": "default-route"
			}
		}]
	}, {
		"name": "recommendation.springistio.svc.cluster.local|http",
		"domains": ["recommendation:8080", "recommendation", "recommendation.springistio:8080", "recommendation.springistio", "recommendation.springistio.svc:8080", "recommendation.springistio.svc", "recommendation.springistio.svc.cluster:8080", "recommendation.springistio.svc.cluster", "recommendation.springistio.svc.cluster.local:8080", "recommendation.springistio.svc.cluster.local", "172.30.209.113:8080", "172.30.209.113"],
		"routes": [{
			"match": {
				"prefix": "/"
			},
			"route": {
				"cluster": "out.recommendation.springistio.svc.cluster.local|http",
				"timeout": "0s"
			},
			"decorator": {
				"operation": "default-route"
			}
		}]
	}]
}
```

Now add a new routerule

```bash
oc create -f istiofiles/route-rule-recommendation-v2.yml
```

The review the routes again

```bash
oc exec $CPOD -c customer -n tutorial curl http://localhost:15000/routes > bfile.json
```

Here is the Before:

```javascript
			"route": {
				"cluster": "out.recommendation.springistio.svc.cluster.local|http",
				"timeout": "0s"
			},
```

and

```javascript
			"decorator": {
				"operation": "default-route"
			}
```

And the After:

```javascript
			"route": {
				"cluster": "out.recommendation.springistio.svc.cluster.local|http|version=v2",
				"timeout": "0s"
			},
```

and

```javascript
			"decorator": {
				"operation": "recommendation-default"
			}
```

If you need the Pod IP

```bash
oc get pods -o jsonpath='{.items[*].status.podIP}' -l app=customer -n tutorial
```

Dive into the istio-proxy container

```bash
oc exec -it $CPOD -c istio-proxy -n tutorial /bin/bash
cd /etc/istio/proxy
ls
cat envoy-rev3.json
```

Snowdrop Troubleshooting

https://github.com/snowdrop/spring-boot-quickstart-istio/blob/master/TROUBLESHOOT.md


# Java (Spring Boot and Vert.x) + Istio on Kubernetes/OpenShift


**Table of Contents**

<!-- toc -->

* [Prerequisite CLI tools](#prerequisite-cli-tools)
* [Setup minishift](#setup-minishift)
* [Setup environment](#setup-environment)
* [Istio installation script](#istio-installation-script)
* [Deploy customer](#deploy-customer)
* [Deploy preference](#deploy-preference)
* [Deploy recommendation](#deploy-recommendation)
* [Updating Redeploying Code](#updating-redeploying-code)
* [Monitoring](#monitoring)
* [Custom Metrics](#custom-metrics)
* [Tracing](#tracing)
* [Istio RouteRule Changes](#istio-routerule-changes)
  * [recommendation:v2](#recommendationv2)
* [Changing Istio RouteRules](#changing-istio-routerules)
    * [All users to recommendation:v2](#all-users-to-recommendationv2)
    * [All users to recommendation:v1](#all-users-to-recommendationv1)
    * [All users to recommendation v1 and v2](#all-users-to-recommendation-v1-and-v2)
    * [Split traffic between v1 and v2](#split-traffic-between-v1-and-v2)
* [Fault Injection](#fault-injection)
  * [HTTP Error 503](#http-error-503)
  * [Delay](#delay)
* [Retry](#retry)
* [Timeout](#timeout)
* [Smart routing based on user-agent header (Canary Deployment)](#smart-routing-based-on-user-agent-header-canary-deployment)
    * [Set recommendation to all v1](#set-recommendation-to-all-v1)
    * [Set Safari users to v2](#set-safari-users-to-v2)
    * [Set mobile users to v2](#set-mobile-users-to-v2)
    * [Clean up](#clean-up)
* [Mirroring Traffic (Dark Launch)](#mirroring-traffic-dark-launch)
    * [Clean up](#clean-up)
* [Access Control](#access-control)
    * [Whitelist](#whitelist)
    * [Blacklist](#blacklist)
* [Load Balancer](#load-balancer)
* [Rate Limiting](#rate-limiting)
* [Circuit Breaker](#circuit-breaker)
  * [Fail Fast with Max Connections and Max Pending Requests](#fail-fast-with-max-connections-and-max-pending-requests)
    * [Load test without circuit breaker](#load-test-without-circuit-breaker)
    * [Load test with circuit breaker](#load-test-with-circuit-breaker)
    * [Clean up](#clean-up)
  * [Pool Ejection](#pool-ejection)
    * [Scale number of instances of `v2` deployment](#scale-number-of-instances-of-v2-deployment)
    * [Test behavior without failing instances](#test-behavior-without-failing-instances)
    * [Test behavior with failing instance and without pool ejection](#test-behavior-with-failing-instance-and-without-pool-ejection)
    * [Test behavior with failing instance and with pool ejection](#test-behavior-with-failing-instance-and-with-pool-ejection)
  * [Ultimate resilience with retries, circuit breaker, and pool ejection](#ultimate-resilience-with-retries-circuit-breaker-and-pool-ejection)







## Circuit Breaker



### Ultimate resilience with retries, circuit breaker, and pool ejection

Even with pool ejection your application doesn't *look* that resilient. That's probably because we're still letting some errors to be propagated to our clients. But we can improve this. If we have enough instances and/or versions of a specific service running into our system, we can combine multiple Istio capabilities to achieve the ultimate backend resilience:
- **Circuit Breaker** to avoid multiple concurrent requests to an instance;
- **Pool Ejection** to remove failing instances from the pool of responding instances;
- **Retries** to forward the request to another instance just in case we get an open circuit breaker and/or pool ejection;

By simply adding a **retry** configuration to our current `routerule`, we'll be able to get rid completely of our `503`s requests. This means that whenever we receive a failed request from an ejected instance, Istio will forward the request to another supposably healthy instance.

```bash
istioctl replace -f istiofiles/route-rule-recommendation-v1_and_v2_retry.yml
```

Throw some requests at the customer endpoint:

```bash
#!/bin/bash
while true
do curl customer-tutorial.$(minishift ip).nip.io
sleep .1
done
```

You won't receive `503`s anymore. But the requests from recommendation `v2` are still taking more time to get a response:

```bash
customer => preference => recommendation v1 from '2039379827-jmm6x': 538
customer => preference => recommendation v1 from '2039379827-jmm6x': 539
customer => preference => recommendation v1 from '2039379827-jmm6x': 540
customer => preference => recommendation v2 from '2036617847-hdjv2': 281
customer => preference => recommendation v1 from '2039379827-jmm6x': 541
customer => preference => recommendation v2 from '2036617847-hdjv2': 282
customer => preference => recommendation v1 from '2039379827-jmm6x': 542
customer => preference => recommendation v1 from '2039379827-jmm6x': 543
customer => preference => recommendation v1 from '2039379827-jmm6x': 544
customer => preference => recommendation v2 from '2036617847-hdjv2': 283
customer => preference => recommendation v2 from '2036617847-hdjv2': 284
customer => preference => recommendation v1 from '2039379827-jmm6x': 545
customer => preference => recommendation v1 from '2039379827-jmm6x': 546
customer => preference => recommendation v1 from '2039379827-jmm6x': 547
customer => preference => recommendation v2 from '2036617847-hdjv2': 285
customer => preference => recommendation v2 from '2036617847-hdjv2': 286
customer => preference => recommendation v1 from '2039379827-jmm6x': 548
customer => preference => recommendation v2 from '2036617847-hdjv2': 287
customer => preference => recommendation v2 from '2036617847-hdjv2': 288
customer => preference => recommendation v1 from '2039379827-jmm6x': 549
customer => preference => recommendation v2 from '2036617847-hdjv2': 289
customer => preference => recommendation v2 from '2036617847-hdjv2': 290
customer => preference => recommendation v2 from '2036617847-hdjv2': 291
customer => preference => recommendation v2 from '2036617847-hdjv2': 292
customer => preference => recommendation v1 from '2039379827-jmm6x': 550
customer => preference => recommendation v1 from '2039379827-jmm6x': 551
customer => preference => recommendation v1 from '2039379827-jmm6x': 552
customer => preference => recommendation v1 from '2039379827-jmm6x': 553
customer => preference => recommendation v2 from '2036617847-hdjv2': 293
customer => preference => recommendation v2 from '2036617847-hdjv2': 294
customer => preference => recommendation v1 from '2039379827-jmm6x': 554
```

Our misbehaving pod `recommendation-v2-2036617847-spdrb` never shows up in the console, thanks to pool ejection and retry.

#### Clean up

```bash
oc scale deployment recommendation-v2 --replicas=1 -n tutorial
oc delete pod -l app=recommendation,version=v2
oc delete routerule recommendation-v1-v2 -n tutorial
istioctl delete -f istiofiles/recommendation_cb_policy_pool_ejection.yml -n tutorial
```

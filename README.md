= Istio Circuit Breaker mission

== Purpose
Showcase Istio's Circuit Breaker via a (minimally) instrumented Eclipse Vert.x microservices.

== Prerequisites
* Openshift 3.9 cluster
* Istio 0.7.1 installed on the aforementioned cluster.
* Enable automatic sidecar injection for Istio (See https://istio.io/docs/setup/kubernetes/sidecar-injection.html[this] 
for details)


== Environment preparation

Create a new project/namespace on the cluster. This is where your application will be deployed.

```bash
oc new-project <whatever valid project name you want>
```

== Build and deploy the application

Execute the following command to build the project and deploy it to OpenShift:
```bash
mvn clean fabric8:deploy -Popenshift
```
Configuration for FMP may be found both in pom.xml and `src/main/fabric8` files/folders.

= Use Cases

== Access the application

* Run the following command to determine the appropriate URL to access our demo. Make sure you access the URL with the 
HTTP scheme. HTTPS is NOT enabled by default:
+
```bash
echo http://$(oc get route istio-ingress -o jsonpath='{.spec.host}{"\n"}' -n istio-system)/breaker/greeting
```
+
You should see an error 404 page (Resource not found), since the application route is not yet exposed.
+
. Apply the `RouteRule` to direct traffic to the greeting and name services:
+
```bash
oc create -f istio/route_rule_public.yml -n $(oc project -q)
```
+
Refresh the page. You should now see the example application.

== Verify application behavior

* Apply the `RouteRule` which is required for the `DestinationPolicy` to take effect:
+
```bash
oc create -f istio/route_rule.yml -n $(oc project -q)
```
. Check that the `RouteRule` is properly applied by executing:
+
```bash
oc get routerule
```
+
* Click on the "Start" button to issue 10 concurrent requests to the name service.
* Click on the "Stop" button to stop the requests
* You can change the number of concurrent requests between 1 and 20.
* All calls go through as expected.


=== Initial Circuit Breaker configuration
* Now apply the initial `DestinationPolicy` that activates Istio's Circuit Breaker on the name service, configuring it to 
allow a maximum of 100 concurrent connections.

+
```bash
oc create -f istio/initial_destination_policy.yml -n $(oc project -q)
```

* You can check that the `DestinationPolicy` is properly applied by executing:

+
```bash
oc get destinationpolicy
```

* Try the application again.


Since we only make up to 20 concurrent connections, the circuit breaker should not trip.

=== Restrictive Circuit Breaker configuration
* Now apply a more restrictive `DestinationPolicy`, after having remove the initial one:
+
```bash
oc delete -f istio/initial_destination_policy.yml
oc create -f istio/restrictive_destination_policy.yml -n $(oc project -q)
```

* Try the application again
+
Since the Circuit Breaker is now configured to only allow one concurrent connection and by default we are sending 10 to 
the name service, we should now see the Circuit Breaker tripping open. However, experimentally, we observe that this does 
not happen in a clear-cut fashion: the circuit is not always open. In fact, depending on how fast the server on which the application is running, the circuit might not break open at all. The reason for this is that is the name service does not do much and thus responds quite fast. This, in turn, leads to not having much concurrency at all.

=== Optional: fault injection

We could try to increase contention on the name service using Istio's fault injection behavior by applying a 1-second delay to 50% of the calls to the name service.

* Delete the original `RouteRule` and apply a new one to do that:
+
```bash
oc delete -f istio/route_rule.yml
oc create -f istio/route_rule_with_delay.yml -n $(oc project -q)
```

* Try the application again
+
You should observe that this does not seem to change how often the circuit breaks open. This is
due to the fact that the injected delay actually occurs between the services. So, in essence, this only time shifts the requests, only increasing concurrency marginally (due to the fact that only 50% of the requests are delayed). This still doesn't let us observe the circuit breaking open properly.

* For more comfort, re-activate the original `RouteRule` since we don't need the artificial one-second delay:
```bash
oc delete -f istio/route_rule_with_delay.yml
oc create -f istio/route_rule.yml -n $(oc project -q)
```

=== Simulate load on the name service

* We need to increase contention on the name service in order to have enough concurrent connections to trip open the circuit breaker. We can accomplish this by simulating load on the name service by asking it to introduce a random processing time. To accomplish this:

* Stop the requests (if that wasn't already the case)
* Checking the "Simulate load" checkbox
* Start the requests.
+
You should now observe the circuit breaking open by observing lots of `Hello, Fallback!` messages.

== Undeploy the application

=== With Fabric8 Maven Plugin (FMP)
```bash
mvn fabric8:undeploy
```

=== With Source to Image build (S2I)
```bash
oc delete all --all
find . | grep openshiftio | grep application | xargs -n 1 oc delete -f
```

=== Remove the namespace
This will delete the project from the OpenShift cluster
```bash
oc delete project <your project name>
```

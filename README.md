### Exposing Fleet Server via Ingress controller and enrolling agents out of Kubernetes

Before start implementing these steps, make sure you are running ECK Operator +1.9

If you are running fleet with ECK and you want to enroll agents out of Kubernetes via ingress controller, you are in the right place.

This doc will guide you to configure the following resources:
Cluster roles / Service Accounts 
Elasticsearch
Kibana
Fleet
Ingress Controller 
These are the resources needed to make it works, however, I am assuming you already have the ingress controller up & running and that you are also running ECK on GCP.

## Implementation

First things first, let's create our elasticsearch cluster by applying the (es.yml)[https://github.com/framsouza/eck-fleet-and-ingress-controller/blob/main/es.yml] manifest, if you take a look closer you will see that there's no specific setting for fleet, the only adjustment need here is the annotation which means the ingress will talk to the elasticsearch resource using https.

Once elasticsearch is up and running, it's time to apply the (kibana.yml)[https://github.com/framsouza/eck-fleet-and-ingress-controller/blob/main/kibana.yml] manifest.
Here you have to specify the following:
```
 config:
	 xpack.fleet.agents.elasticsearch.host: "https://cluster1-es-http.default.svc:9200"
	 xpack.fleet.agents.fleet_server.hosts: ["https://fleet-server-agent-http.default.svc:8220"]
```

`xpack.fleet.agents.elasticsearch.host` must point to elasticsearch cluster where the elastic agent will ship the data. `xpack.fleet.agents.fleet_server.hosts` must point to Fleet Server that Elastic Agent should connect to. (Once you apply the fleet manifest, the fleet svc will be automatically created)

I am changing the URL path to access kibana via `/kibana`, to kibana be aware this is behind an ingress we need to specify the following configuration:
```
 server:
	 rewriteBasePath: true
	 basePath: "/kibana"
	 publicBaseUrl: "https://framsouza.co/kibana"
```
The ingress controller will also communicate with Kibana using HTTPS, to do so we need to also specify the annotations the same we did with ES.
```
 http:
	 service:
	 metadata:
	 annotations:
	 cloud.google.com/app-protocols: '{"https":"HTTPS"}'
	 service.alpha.kubernetes.io/app-protocols: '{"https":"HTTPS"}'
```

Last but not least, as we changed the kibana entry point, we also need to adjust the readinessProbe,
We are now using `/kibana` which means the healthy check must run against `/kibana/login` from now on.

Once kibana is up and running and properly connected with ES, we are ready to deploy fleet.
But before that, make sure you applied the (clusterroles.yml)[https://github.com/framsouza/eck-fleet-and-ingress-controller/blob/main/clusterroles.yml] manifest to make sure you have the proper permissions set.

If you check the (fleet.yml)[https://github.com/framsouza/eck-fleet-and-ingress-controller/blob/main/fleet.yml] manifest, you will see the same annotations as I mentioned before for ES & Kibana. The part that matter for us here is 
```
 containers:
	 - env:
	 - name: KIBANA_FLEET_HOST
	 value: https://kibana-cluster1-kb-http.default.svc:5601/kibana
	 name: agent
```
Which will tell fleet to connect into Kibana using the `/kibana`

Once you apply it, make sure to check the fleet logs to check if the connection with elasticsearch & kibana was established.

You can easily use the following commands to check the status of the resources:
`kubectl get es` , `kubectl get kibana`, `kubectl get agents`

You should see the green status for all of them.
Now it's time to deploy the ingress manifest, I am using the following:
```
 - path: /kibana
	 pathType: Prefix
	 backend:
	 service:
	 name: kibana-cluster1-kb-http
	 port:
	 number: 5601
	 - path: /
	 pathType: Prefix
	 backend:
	 service:
	 name: fleet-server-agent-http
	 port:
	 number: 8220
```
This means that I should be able to access Kibana by https://framsouza.co/kibana and to enroll the agents I am using /.

We have some limitations to using `/fleet` to access fleet server at the moment and the workaround to make it works is to use the root source to enroll the agents that are running out of Kubernetes.

Once you apply the ingress, you should be able to access Kibana and see the following on the Fleet page:



Great! It means you have Fleet Server up and running and now you are ready to enroll agents that is running out of Kubernetes using https://framsouza.com/
For this example, I enrolled my own laptop running the following command:

sudo ./elastic-agent install -f --url=https://framsouza.co/ --enrollment-token=<TOKEN> --insecure

You should be able to grab your token by clicking on "Add agent" at the Fleet page.

If you access the Fleet page again, you will see you have one more agent available with metrics and logs.




If you have any question, please do not hesitate to reach out via email fram.souza14@gmail.com

Cheers.

Title: Akka Cluster On Kubernetes
Date: 2016-05-17 21:10
Category: Programming
Tags: Akka, Actors, Scala, Kubernetes

One of the challenges of running an Akka cluster application is the bootstrapping step required to discover other nodes in the cluster. This post illustrates how to make use of Kubernetes headless services to deploy an Akka cluster that automatically discovers its peers.

Normally, Kubernetes services resolve to a single IP address belonging to one of the child containers that match the selection criteria. A headless service, on the other hand, return a list of all the IP addresses that are under its watch. This is achieved by setting the `clusterIP` field to `None`.

```yaml
apiVersion: v1
kind: Service
metadata:
    name: discovery-svc
    labels:
        name: discovery-svc
        app: my-akka-app
spec:
    clusterIP: None
    ports:
        - name: discovery-port
          port: 2600
    selector:
        name: cluster-node
```

Next we define a deployment spec for the cluster application itself. The following spec sets the initial cluster size to three nodes. The crucial bit is the definition of an environment variable named `DISCOVERY_SERVICE` which is set to the value of the Kubernetes cluster DNS entry for the service we defined earlier  (on Google Container Engine this is usually the name of the service followed by `default.svc.cluster.local`). Please note that typically service information is exposed by Kubernetes as environment variables of the form `<normalised_service_name>_SERVICE_HOST` and `<normalised_service_name>_SERVICE_PORT` (in this example, the variables would be `DISCOVERY_SVC_SERVICE_HOST` and `DISCOVERY_SVC_SERVICE_PORT`). However, I had trouble accessing these variables from within my containers -- hence the need to define an explicit variable with the hard-coded service DNS entry.

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
    name: cluster-dep
    labels:
        name: cluster-dep
        app: my-akka-app
spec:
    replicas: 3
    template:
        metadata:
            name: cluster-node
            labels:
                name: cluster-node
                app: my-akka-app
        spec:
            terminationGracePeriodSeconds: 60
            containers:
                - image: my-docker-registry.com/my-akka-app:1.0
                  imagePullPolicy: Always
                  name: cluster-node
                  ports:
                    - name: cluster-port
                      containerPort: 2600
                  env:
                    - name: DISCOVERY_SERVICE
                      value: discovery-svc.default.svc.cluster.local
                  livenessProbe:
                    tcpSocket:
                      port: 2600
                    initialDelaySeconds: 15
                    timeoutSeconds: 2
```

In order for the discovery to work, we need to bootstrap the application as follows:

```scala
def main(args: Array[String]): Unit = {
  val conf = resolveConfig("MyCluster", 2600)
  val actorSystem = ActorSystem("MyCluster", conf)
  // rest of the init code
}

private def resolveConfig(actorSystemName: String, port: Int): Config = {
  val hostAddress = getHostAddress
  val seedNodes = getSeedNodes(hostAddress, port)

  ConfigFactory.empty()
    .withValue("akka.cluster.seed-nodes", ConfigValueFactory.fromIterable(seedNodes.map(node => s"akka.tcp://${actorSystemName}@$node")))
    .withValue("akka.remote.netty.tcp.hostname", ConfigValueFactory.fromAnyRef(hostAddress))
    .withValue("akka.remote.netty.tcp.port", ConfigValueFactory.fromAnyRef(port))
    .withFallback(ConfigFactory.load())
    .resolve()
}

private def getSeedNodes(hostAddress: String, port: Int): Seq[String] = {
    // try to resolve the list of IP addresses from the discovery service or return local address
    val seedNodes = Option(System.getenv("DISCOVERY_SERVICE")).fold(Seq.empty)(InetAddress.getAllByName(_).map(addr => s"${addr.getHostAddress}:$port"))
    if(seedNodes.isEmpty){
      Seq(s"$hostAddress:$port")
    } else {
      seedNodes
    }
}

private def getHostAddress: String = {
  NetworkInterface.getNetworkInterfaces
    .find(_.getName equals "eth0")
    .flatMap { interface =>
      interface.getInetAddresses.find(_.isSiteLocalAddress).map(_.getHostAddress)
    }
    .getOrElse("127.0.0.1")
}
```

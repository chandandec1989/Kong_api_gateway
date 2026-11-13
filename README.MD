# Kong Installation
Kong comes with a wide range of installation options. Some of them are directly supported by kong, while few other options are supported by the community. Look at different installation options here. I personally use docker based deployment because of its portable. Let's look at a single node kong set up with PostgreSQL.

## kong medium link:-

```
https://medium.com/swlh/kong-api-gateway-zero-to-production-5b8431495ee

https://medium.com/@claudioacquaviva/kong-ai-mcp-gateway-and-kong-mcp-server-technical-breakdown-13420f610ee6

https://medium.com/@nanditasahu031/a-comprehensive-guide-to-kong-api-gateway-11cc374c1ce5

https://medium.com/@claudioacquaviva/multi-llm-react-ai-agent-with-kong-ai-gateway-3-10-and-langgraph-7429c9c1f5ee

```

## Step 1: Create a docker network to deploy kong and PostgreSQL with connectivity

```
docker network create kong-net
```

## Step 2: Start PostgreSQL in the “kong-net” network

```
docker rm kong-database = delete database if required
```

```sh
docker run -d --name kong-database --network=kong-net -p 5432:5432 \
  -e POSTGRES_USER=kong -e POSTGRES_DB=kong -e POSTGRES_PASSWORD=kong \
  postgres:16
```

## Step-3:- Check the database container:

```
docker ps -a --filter name=kong-database
docker logs kong-database
docker exec kong-database pg_isready -U kong
```

## Step 4: Run migration script on the Postgres DB and get it ready for Kong

Wait until PostgreSQL is accepting connections:

```sh
until docker exec kong-database pg_isready -U kong; do sleep 1; done
```

```sh
docker run --rm --network=kong-net \
  -e KONG_DATABASE=postgres \
  -e KONG_PG_HOST=kong-database \
  -e KONG_PG_PASSWORD=kong \
  kong:latest kong migrations bootstrap
```

## Step 5: Start the actual Kong docker

```sh
docker run -d --name kong --network=kong-net \
  -e KONG_DATABASE=postgres \
  -e KONG_PG_HOST=kong-database \
  -e KONG_PG_PASSWORD=kong \
  -e KONG_PROXY_ACCESS_LOG=/dev/stdout \
  -e KONG_ADMIN_ACCESS_LOG=/dev/stdout \
  -e KONG_PROXY_ERROR_LOG=/dev/stderr \
  -e KONG_ADMIN_ERROR_LOG=/dev/stderr \
  -e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" \
  -p 8000:8000 -p 8443:8443 \
  -p 127.0.0.1:8001:8001 -p 127.0.0.1:8444:8444 \
  kong:latest
```

## You can check if kong is up and running with the following command.

```
curl -i http://localhost:8001/
```

# Kong For Kubernetes
If all of your APIs exist within a Kubernetes cluster, the best way to deploy kong is by using the Kong Ingress controller. This installs a couple of containers in a pod, one acting as a control pane and other the data pane. The advantage of this model is that it's Kubernetes-native and automatically discovery APIs through the API server.

Installing kong into Kubernetes is pretty simple. Run the below kubectl.

```
kubectl apply -f https://bit.ly/kong-ingress-dbless
```

This will setup kong with ETCD persistence, few CRD, and a POD with both controller and kong proxy.

## Kong Configuration
Lets quickly understand few terminologies needed to configure kong

## Service — 
The kong object that binds the upstream API with kong

```
curl -i -X POST — url http://localhost:8001/services/ — data ‘name=example-service’ — data ‘url=http://mockbin.org'
```

In the above example, we create a service named “example-service” pointing to an upstream API “http://mockbin.org”. “http://localhost:8001/services/” is the kong admin endpoint to create a service object

## Route — 
The kong object that binds a service with route path for the API consumers

```
curl -i -X POST — url http://localhost:8001/services/example-service/routes — data ‘hosts[]=example.com&paths[]=/mockbin’
```

In the above example, we create a route for the “example-service” service pointing to the path “/mockbin”. “http://localhost:8001/services/{service-name}/routes” is the kong admin endpoint to create a route object under the specific service.

Now hit http://localhost:8000/mockbin/, to see if the proxy is working. There are many nuances to configure APIs. See detailed document at doc.

If you are using Kong for Kubernetes, the technologies remain the same. But the configuration will be a Kubernetes custom resource YAML.


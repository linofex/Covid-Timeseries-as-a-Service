
A decentralized framework for the distribution of lambda functions, in a Function-as-a-Service (FaaS) model, to multiple serverless platforms.

The reference system is depicted in the figure below.

![](docs/arch.png)

- The _clients_ are end-user applications wishing to execute stateless tasks.
- The _serverless platforms_ are systems that can execute tasks upon receiving a lambda function execution request. It is not required that these platforms interact with one another.
- The _brokers_ are the entry point of clients in the system: they forward every incoming lambda function to the serverless platform that is deemed to be the best choice at any given point in time. Like serverless platforms, the brokers do not communicated with another. However, they must be notified of the existence of serverless platforms (and which lambda functions they can use), which can be done by a so-called _edge controller_ not depicted in the figure above.

The components communicate with one another via Google's [gRPC](https://grpc.io/).
An alternative, there is experimental support of [QUIC](https://quicwg.org/) using Facebook's [proxygen](https://github.com/facebook/proxygen) and [mvfst](https://github.com/facebookincubator/mvfst).

## Components

For each of the components there are different versions, illustrated below.

###  Broker

- `edgerouter`: implements the e-router concept described [here](https://ccicconetti.github.io/cloudcom2018.html), where every broker has a table of possible destinations with an associated weight corresponding to the response delay, and the next destination is selected according to one of several policies.
- `edgedispatcher`: implements the dispatcher concept described [here](https://ccicconetti.github.io/percom2019.html), where the broker selects the destination that minimizes the response delay, based on a predictions.

### Serverless platform

- `edgecomputer`: simulates a serverless platform with given characteristics (CPU speed, memory, number of containers, etc) and task properties (e.g., memory and operations requested)/
- `edgecomputerfacerec`: performs face/eyes detection in 2D images using [OpenCV](https://opencv.org/)/
- `edgecomputerwsk`: acts as proxy towards an Apache OpenWhisk platform hosted elsewhere, see below.

### Client

- `edgeclient`: continually executes lambda functions with a given name and content; the intertime between consecutive requests can be drawn from uniform or exponential random variables; multiple threads can be spawn with same characteristics.
- `edgemulticlient`: create a pool of clients issuing lambda requests in accordance with a timetable generated by `clienttracegen`.
- `edgeippclient`: create a pool of clients issuing lambda requests with an ON/OFF pattern with given characteristics.
- `facerecclient`: requests detection of face (optionally also eyes) in pictures; can use a real-time stream from camera.
- `wskproxy`: acts as a proxy from Apache OpenWhisk clients (only for actions), see below.


## Integration with ETSI MEC

[ETSI MEC](https://www.etsi.org/technologies/multi-access-edge-computing) is a set of specifications, under definition in the Industry Study Group MEC of the European Telecommunications Standards Institute, aimed at defining a reference architecture and open APIs for the deployment and lifecycle management of vendor-neutral applications in edge networks of telecom operators.

We have implemented a partial integration of our _Serverless on Edge_ framework with the ETSI MEC, specifically with the `Mx2` interface. More information can be found in the [dedicated GitHub repository](https://github.com/ccicconetti/etsimec), which is added as a sub-module to this one.

This project is listed as one of the MEC Solutions in the [ETSI MEC ecosystem wiki page](https://mecwiki.etsi.org/index.php?title=MEC_Ecosystem).

## Integration with Apache OpenWhisk

[Apache OpenWhisk](https://openwhisk.apache.org/) is a distributed serverless platforms that realizes the FaaS model by executing functions in response to events of different types, called actions, triggers, and feeds.

We have integrated partially our _Serverless on Edge_ framework by means of two components, called `edgecomputerwsk` and `wskproxy`, which allow actions issued by clients to be executed on OpenWhisk platforms. 

If interested check out more details and instructions [here](docs/openwhisk_integration.md).

## Building

See [building instructions](docs/BUILDING.md).

## Example

After having built the framework, you may start by deploying the following example (the numbers represent the listening TCP ports).

![](docs/example.png)

Let us ignore for the moment the gray components, i.e., `edgecomputerclient` and `forwardingtableclient`.

You have to open 5 shells and execute the following commands in each, assuming that all the shells have been opened on the same host, there are no other applications bound to these ports, and that the current working directory contains the executables, e.g., if you have build in debug it is `build/debug/Executables` from the git repo root.

First, launch the controller, which will listen to `6475`:

```
./edgecontroller --server 127.0.0.1:6475
```

Then launch the two edge computers (with default characteristics, which creates two lambda functions `clambda0` and `glambda0` executed on containers with different CPU speed) and connect them to the controller:

```
./edgecomputer --server 127.0.0.1:10000 \
               --utilization 127.0.0.1:20000 \
               --controller 127.0.0.1:6475
```

and

```
./edgecomputer --server 127.0.0.1:10001 \
               --utilization 127.0.0.1:20001 \
               --controller 127.0.0.1:6475
```

Then launch the edge router and connect it to the controller, as well:

```
./edgerouter --server 127.0.0.1:6473 \
			   --configuration 127.0.0.1:6474 \
			   --controller 127.0.0.1:6475
```

At this point the system is up and ready and accepts execution of lambda functions both directly on the edge computer (if issues to end-points on port 10000 or 10001) and on the edge router (at port 6473). We then launch a client and ask to execute 10 tymes the same function `clambda0` with size 10000 bytes every second:

```
./edgeclient --lambda clambda0 \
		      --server 127.0.0.1:6473 \
		      --inter-request-time 1 \
		      --max-requests 5
```

An example of output is the following:

```
1.11078 0.109332 0 127.0.0.1:10001 clambda0 2
2.11141 0.107049 0 127.0.0.1:10000 clambda0 2
3.11257 0.110859 0 127.0.0.1:10001 clambda0 2
4.10905 0.105978 0 127.0.0.1:10000 clambda0 2
5.11009 0.107862 3 127.0.0.1:10000 clambda0 2
I0226 14:48:27.862442 265811392 edgeclientmain.cpp:188] latency 0.108216 +- 0.00171617
I0226 14:48:27.863188 265811392 edgeclientmain.cpp:191] processing 0.1058 +- 0.00172047
```

The output reports:

- a timestamp, in seconds;
- the response time, in seconds;
- the load reported by the edge computer, in percentage;
- the end-point of the edge computer that has executed the lambda function: you may notice that the edge route is balancing between the two edge computers;
- the lambda function name;
- the number of hops to reach the edge computer (client -- edge router + edge router -- edge computer).

For all components verbosity can be increased by specifing the environment variable `GLOG_v=X`, with `X` 1 or greater, since [glog](https://github.com/google/glog) is used for logging.

The current state of the weights in the edge routers can be queried (via gRPC) as follows:

```
./forwardingtableclient --server 127.0.0.1:6474
```

Example of ouput:

```
Table#0
clambda0 [0.106305] 127.0.0.1:10000 (F)
         [0.108944] 127.0.0.1:10001 (F)
glambda0 [1       ] 127.0.0.1:10000 (F)
         [1       ] 127.0.0.1:10001 (F)
Table#1
clambda0 [1] 127.0.0.1:10000 (F)
         [1] 127.0.0.1:10001 (F)
glambda0 [1] 127.0.0.1:10000 (F)
         [1] 127.0.0.1:10001 (F)
```

The current load of the edge computers can be queried (again, via gRPC) as follows, for instance on the first edge computer:

```
./edgecomputerclient --server 127.0.0.1:20000
```

Example of output while idle:

```
0.60116 bm2837 0.000000
0.601225 arm 0.000000
1.60027 bm2837 0.000000
1.6003 arm 0.000000
2.60552 bm2837 0.000000
2.60557 arm 0.000000
```

Note that there are two lines per second, since in its default configuration the edge computer simulates two CPUs (called `arm` and `bm2837`), each hosting one container. By issuing `clambda0` only `arm` is used.

Example of output while running an edge client as before:

```
0.35004 bm2837 0.000000
0.350097 arm 0.107947
1.35395 bm2837 0.000000
1.35399 arm 0.133199
2.35423 bm2837 0.000000
2.35427 arm 0.128262
```

The same example can be executed, for instance, launching:

- `edgedispatcher` instead of `edgerouter`;
- `OpenCV/facerecclient` instead of `edgeclient` and `edgecomputerfacerec` instead of `OpenCV/edgecomputer`, though the latter requires to download suitable models from [OpenCV repositories](https://github.com/opencv).

## Credits

- Chiara Spinelli ([twitter](https://twitter.com/chiarapeggy) / [instagram](https://www.instagram.com/chiarapeggy/)): logo artwork


Cloud Computing Project
OpenStack Project: Timeseries-as-a-Service
This project has been developed on a cluster of VMs hosted on University servers.
The project specification can be found in [..]
We decided to build a simple GUI in order to show the Italy Covid-related data (both for each single Region and the sum aggregate for Italy-related data) provided by Protezione Civile until a certain day; subsequent data have been randomly produced.
The application consists in a container which runs a Flask server, this component is responsible to build a simple REST API in order manage HTTP requests issued by a producer. These requests are handled through an interaction with a Gnocchi DB hosted in another container.
Since the project was meant to test the understanding of the OpenStack mechanisms acquired during the cloud computing course, all the IP address used inside the files are static so it is difficult to provide the way to launch the program.
For a better understanding of the functioning we try to provide some information inside this README.md file and inside the code using comments.
Both the Producer and the Consumer have been created through the OpenStack Horizon Dashboard and  have been inserted in the admin domain so that they have privileges that allow them to retrieve the list of the metrics stored in Gnocchi.

The Consumer is mainly responsible to create a REST application using Flask (the main function only runs the Flask server). The flask server contains an handlers collection in order to manage all the HTTP Requests issued by the producer to given endpoints (each handler starts with @app.route('endpoint)).
Two fundamental statements used inside the code are the following:
-   redirect(url_for(endpoint)): issues an HTTP request to a given endpoint
-   render_template(pagina.html): updates the GUI, the HTML code is located inside the directory [..]

Gnocchi:
Gnocchi [https://gnocchi.xyz/] stores metrics and resources and provides a REST API in order to manipulate them. In this project we stored only metrics: each Region is a metric, this is created inside the Gnocchi DB and stored on Ceph according to a certain archive-policy.
Gnocchi does not store each single datum but instead stores aggregates: an archive-policy defines the lifespan, the granularity and the aggregate type to be stored for a certain metric.
In this case we defined a new archive-policy called years_rate with timespan equal to one year, granularity equal to one day and sum as aggregate type.

The Gnocchi container has been built through the command (please pay attention to the final dot in order to specify the current directory)
docker build -t docker_gnocchi .

Dockerfile
-	FROM: specifies the image from the central repository to be used, alpine is a Python lightweigth version
-	RUN: specifies the command to be executed inside the container at installation time (in this case we need to create a directory in which we insert the requirements.txt file containing the requirements to be installed through pip3)
-	EXPOSE: in order to publicly expose a service running inside a container to external networks we need to perform 2 steps: 
o	The first one is to to configure a container in order to expose a service through a certain UDP/TCP port. In this case in order to receive http requests we need to expose the service on port 8080; 
o	The second step consists in mapping the chosen port in one of the public ports of the public IP address of the system which is hosting the server. For this reason the container is launched with the command: 
docker run -p 5000:8080 -it server-consumer 
This command tells the container to execute the docker image and to open a shell inside the container once the program will be executed.
-	ENTRYPOINT, CMD: these fields allow to specify the command to be executed when the container is started and could have been done also through the command
CMD ["/usr/bin/python3", "/root/my_program.py"]
  
In order to stop a container:
•	Firstly, you need to get the running container list through the command docker ps  
Then you need to launch the command docker stop ID
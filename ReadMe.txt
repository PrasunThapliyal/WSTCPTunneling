27 Apr 2023
===========
WSTCPTunnel

This would be a POC.
The idea is to use Websockets to tunnel TCP (/UDP) traffic.

Inspiration
-----------
	A typical microservices application could have several services, databases, messaging apps (Kafka) etc, with complex dependencies among those apps.
	For eg, ETP requires Cassandra, Postgres, Kafka, Pithos (if on-prem), AWS S3 (if cloud), PM, Tron, P2Auth, SFDataProvider, Contact, GCS, Commissioning, ODV etc
	Each of these apps would futher have their Southbound dependencies
	
	Now, how do we debug ETP. 
	
	Since it is REST based, one option is to use Postman. But the application overall is ever changing, new APIs get added .. and maintining these postman collections is not the best of options
	The other option is to run all these apps locally (or the minimum that is required to debug that particular use case)
		For this second options, we've already done a setup where we run ETP, SB and Postgres locally, Pithos and Cassandra on Onxv and rest of the apps on a Dev server.
		We wrote a HTTP Proxy server, and were able to use the UI from Dev to integrate with ETP and SB in our local machine. This covers a significant portion of our app, but not all.
		
	Another possible solution that could have been, sans our corporate firewall, would be to redirect calls from our Load Balancer (HAProxy) running in Dev to call ETP on our localhost instead of the ETP running in the cluster. That would help in the traffic comming towards ETP. For traffic going out from ETP (to every southbound), if Dev were onxv, we could simply define port forwarding and that would solve all our goals.
	But for the firewall ..
	
	If we could get to run ETP in our Visual Studio while rest of the apps run on some Dev server, and somehow bypass firewall, that would be great !!
	
Plan
----
	0. Note that our corporate firewall only allows HTTP traffic to go out from our machines.
	1. Create a local proxy server. Here we need to extend our HTTPProxy to cover websockets.
	2. Once we have Websocket running, we'll use its binary serializer to send/receive TCP data. 
		For eg, if we want to connect to Postgres on Dev on port 5432
		Tell ETP that postgres is on localhost:7000.
		Run a TCP Server on localhost:7000. (Call it A)
		At the other end of Websocket (on Dev machine), run a TCP client (call it B) that would connect to Postgres on Dev:5432
		When ETP does a connect it connects to A. A will send a custom message on WS that would intruct B to do a connect with Dev:5432
		Once both these connections are established, ETP and Postgres would each wan to send/receive data as per the DB protocol.
		Whenever ETP does a send, send the bytes over WS. At the other end B would receive this data and send to Postgres. Likewise Postgres to ETP.
		
Development stages
------------------
	1. First lets not worry about WS or the dev server.
	2. ETP and Postgres running locally.
	3. Write A and B in the same application. Let ETP talk via A -> B to Postgres.
	4. Once this is working, Write a WS client and WS server, and try on the local setup
	5. Then move to Dev
	6. Then extend it to all apps like Kafka, Cassandra etc
	7. Then extend for multiple users
	
Sequence diagram using https://www.websequencediagrams.com/
--------------------------------------
	title WS Tunnel

	ETP->TCPProxyServer(A): Connect to Postgres on Dev
	note over TCPProxyServer(A): A and WSClient are in the same application
	TCPProxyServer(A) -> WSClient: Custom mesage instructing B to connect to Postgres
	WSClient -> WSServer: Custom mesage instructing B to connect to Postgres
	note over TCPProxyClient(B): WSServer and B are in the same application
	WSServer -> TCPProxyClient(B): Custom mesage instructing B to connect to Postgres
	TCPProxyClient(B) -> Postgres: Connect


	ETP->TCPProxyServer(A): Send to Postgres on Dev
	TCPProxyServer(A) -> WSClient: Custom mesage instructing B to send to Postgres
	WSClient -> WSServer: Custom mesage instructing B to send to Postgres
	WSServer -> TCPProxyClient(B): Custom mesage instructing B to send to Postgres
	TCPProxyClient(B) -> Postgres: Send


	Postgres -> TCPProxyClient(B): Send
	TCPProxyClient(B) -> WSServer: Custom mesage instructing A to send to ETP
	WSServer -> WSClient: Custom mesage instructing A to send to ETP
	WSClient -> TCPProxyServer(A): Custom mesage instructing A to send to ETP
	TCPProxyServer(A) -> ETP: Send to ETP

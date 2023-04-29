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

==============================================================

Well I got stuck at Step 3 of this
	1. First lets not worry about WS or the dev server.
	2. ETP and Postgres running locally.
	3. Write A and B in the same application. Let ETP talk via A -> B to Postgres.
	
Both sides connected succesfully (as in TCP connect), but as soon as bytes began to flow, the client raised an exception stating that there's a possible Man-in-the-middle attack.
This was due to lack of SSL in my communication, and I haven't been able to figure out yet.

However, a different wonderful thing was discovered as I was searching to solve the SSL exception.

ref: This and a few other links have the same technique
	https://stackoverflow.com/questions/15768913/relay-postgresql-connection-over-another-server
	
(****) Basically, if I can reach the Dev machine directly via SSH, I can setup what is known as ssh tunneling

	CIENA+pthapliy@DESKTOP-V99LATL MINGW64 /C/GIT/hmaccsharp (PR_01)
		$ ssh bpadmin@carbonundoredo.test.apps.ciena.com
		bpadmin@carbonundoredo.test.apps.ciena.com's password:
		Last login: Fri Apr 28 07:00:49 2023 from 10.124.6.203
		[bpadmin@ip-10-78-133-253 ~]$ 

	Further, on the Dev Server, note down that to reach the AWS RDS (Postgres), I have to call psql -h 10.78.133.10 -U bpadmin postgres
	This is a private IP and can only be reached from within the Dev server.
	Note down the IP/Username/Password of the postgres running on Dev (actually not running on dev -- this one is RDS, but reachable through Dev)
	
	Next, pick a random new port number that your could-have-been proxy would have supplied. And run this command on the bash terminal in your local laptop
	I chose port 7654
	
	CIENA+pthapliy@DESKTOP-V99LATL MINGW64 /C/GIT/hmaccsharp (PR_01)
		$ ssh bpadmin@carbonundoredo.test.apps.ciena.com -CNL localhost:7654:10.78.133.10:5432
		bpadmin@carbonundoredo.test.apps.ciena.com's password:

	After you enter the password, the command hangs .. let it be .. don't close the window
	
	What we're saying here is that a postgres client when pointed to locahost:7654 will actually connect to Dev->10.78.133.10:5432
	That's it ..
	
	Open a new CMD window
	
	C:\Users\pthapliy>cd "C:\Program Files\PostgreSQL\14\bin"
	
	C:\Program Files\PostgreSQL\14\bin>psql.exe -h localhost -p 7654 -U bpadmin postgres
		Password for user bpadmin:
		psql (14.5, server 11.17)
		WARNING: Console code page (437) differs from Windows code page (1252)
				 8-bit characters might not work correctly. See psql reference
				 page "Notes for Windows users" for details.
		SSL connection (protocol: TLSv1.2, cipher: AES128-SHA256, bits: 128, compression: off)
		Type "help" for help.

		postgres=> \c topologyplanner
		
	And just like that, we've connected to the RDS on Dev machine.
	





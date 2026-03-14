Since you are now at poll() event loop, the previous step (Part 2) is the foundation: building a simple TCP proxy that forwards data between a client and a backend server.
This is the core idea of a load balancer.

I’ll explain slowly and clearly, since you prefer simple step-by-step explanations.

Load Balancer Project – Part 2
Building a Simple TCP Proxy

Goal:

Client ----> Load Balancer ----> Backend Server

The load balancer will:

Accept client connection

Connect to backend server

Forward client data → backend

Forward backend response → client

So the load balancer becomes a middleman.

1. Network Flow

Example request:

curl localhost:8080

Flow:

Client
   |
   v
Load Balancer (8080)
   |
   v
Backend Server (9001)

Step by step:

Client -> LB : HTTP request
LB -> Backend : forward request
Backend -> LB : response
LB -> Client : forward response
2. Basic Server Socket

Load balancer listens for clients.

int listenfd = socket(AF_INET, SOCK_STREAM, 0);

Bind to port 8080

sockaddr_in addr{};
addr.sin_family = AF_INET;
addr.sin_port = htons(8080);
addr.sin_addr.s_addr = INADDR_ANY;

bind(listenfd, (sockaddr*)&addr, sizeof(addr));

Start listening

listen(listenfd, 128);

Now server waits for connections.

3. Accept Client Connection
int client_fd = accept(listenfd, NULL, NULL);

Now:

client_fd -> connected to client
4. Connect to Backend Server

We create another socket to connect to backend.

Backend example:

127.0.0.1:9001

Create socket:

int backend_fd = socket(AF_INET, SOCK_STREAM, 0);

Set address:

sockaddr_in backend{};
backend.sin_family = AF_INET;
backend.sin_port = htons(9001);

inet_pton(AF_INET, "127.0.0.1", &backend.sin_addr);

Connect:

connect(backend_fd, (sockaddr*)&backend, sizeof(backend));

Now:

client_fd <----> load balancer <----> backend_fd
5. Data Forwarding

Now the LB must forward data both directions.

Two flows exist:

client -> backend
backend -> client
Client → Backend

Read client request.

char buf[4096];

int n = read(client_fd, buf, sizeof(buf));

Send to backend.

write(backend_fd, buf, n);
Backend → Client

Read backend response.

int n = read(backend_fd, buf, sizeof(buf));

Send back to client.

write(client_fd, buf, n);
6. Full Simple Proxy Loop

Simplified version:

while (true) {

    char buf[4096];

    int n = read(client_fd, buf, sizeof(buf));
    if (n <= 0) break;

    write(backend_fd, buf, n);

    n = read(backend_fd, buf, sizeof(buf));
    if (n <= 0) break;

    write(client_fd, buf, n);
}

This forwards request and response.

7. What This Proxy Does

Example request:

curl localhost:8080

Internally:

curl -> LB (8080)
LB -> Backend (9001)
Backend -> LB
LB -> curl

Client thinks it is talking to the server, but actually:

client <-> proxy <-> server
8. Testing Backend

Run a backend server:

python3 -m http.server 9001

Start load balancer.

Test:

curl localhost:8080

Response comes from port 9001 server.

9. Problem With This Version

This proxy works but has big limitations.

Problem 1

Only one client at a time.

client1 -> works
client2 -> must wait
Problem 2

Blocking IO.

read()
write()

If backend is slow:

whole server blocks
Problem 3

Cannot handle many connections.

Modern load balancers handle:

10k
100k
1M connections
10. Why We Introduced poll() (Part 3)

To solve blocking problem we use IO multiplexing.

Instead of:

wait for one socket

We wait for many sockets.

Example:

poll([
client1,
client2,
client3,
backend1,
backend2
])

Then kernel tells us:

socket ready to read
socket ready to write

This creates an event loop server.

Architecture becomes:

while(true)
{
    poll(all sockets)

    handle ready sockets
}

This is exactly how:

NGINX

Redis

HAProxy

Node.js

work internally.

11. Architecture Progress So Far
Part 1

Basic TCP server

client -> server
Part 2

Proxy

client -> proxy -> server
Part 3

Event-driven proxy

poll()
multiple connections
Next (Part 3 in Detail)

Next we go deeper into:

poll() Event Loop Architecture

We will build:

connection struct
fd → connection map
pollfd vector
event loop

This is the same architecture used in Redis networking layer.

If you want, I can also show a complete 150-line minimal C++ load balancer that already supports 1000+ concurrent connections.

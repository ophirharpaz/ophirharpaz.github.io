---
layout: post
title:  "How Does SSH Port Forwarding Work?"
image: https://ophirharpaz.github.io/images/ssh_port_forwarding.png
vertical: Code
excerpt: "Explanation of how SSH (local) port forwarding is implemented in DropBear SSH (and probably in others)"
---

I often use the command `ssh server_addr -L localport:remoteaddr:remoteport` to create an SSH tunnel. 
This allows me, for example, to communicate with a host that is only accessible to the SSH server.
You can read more about SSH port forwarding (also known as "tunneling") [here](https://help.ubuntu.com/community/SSH/OpenSSH/PortForwarding).
This blog post assumes this knowledge.

But what happens behind the scenes when the above-mentioned command is executed? 
What happens in the SSH client and server when they respond to this port forwarding instruction?

In this blog post, I'll focus on the [*DropBear SSH*](https://github.com/mkj/dropbear) 
implementation and also stick to *local* (as opposed to *remote*) port forwarding. I am *not* describing my personal
research process here, because there's already enough information to share. Believe me ;)

## TL;DR

This section summarizes the process without quoting any line of code. 
If you wish to understand how local port forwarding works in SSH, without going into any specific implementation,
this section will definitely suffice.

![SSH Local Port Forwarding in a Diagram](/images/ssh_port_forwarding.png "SSH Local Port Forwarding in a Diagram")

1. The client creates a **socket** and binds it to *localaddr* and *localport* (actually, it binds a socket for each 
address resolved from *localaddr*, but let's keep things simple). 
If no *localaddr* is specified (which is usually the case for me), the client will create a socket for *localhost*. 
`listen()` is called on the created, bound socket.
2. Once a connection is accepted on the socket, the client creates a **channel** with the socket's file descriptor 
(*fd*) for reading and writing. Unlike sockets, channels are part of the SSH protocol and are not operating-system objects.
3. The client then sends a **message** to the server, informing it of the new channel. The message includes the client's channel 
identifier (index), the local address & port and the remote address & port to which the server should connect later on.
4. When the server sees this special message, it creates a new **channel**. It immediately "attaches" this channel to the 
client's one, using the received identifier, and sends its own identifier to the client. This way the two sides exchange channel IDs for
future communication.
5. Then, the server connects to the remote address and port which were specified in the client's payload. 
If the connection succeeds, its **socket** is assigned to the server's channel for both reading and writing.
6. Data is sent and received between the sides whenever `select()`, which is called from the session's main loop, 
returns file descriptors (sockets) that are ready for I/O operations.

**So just to make things clear:**

- On the client side — a socket is connected to the local address, which is where data is read from 
(before sending to the server) or written to (after being received from the server).
- On the server side, there is another socket which is connected to the remote address. 
This is where data is sent to (from the client) or received from (on its way to the client).

## Drill-Down

What follows is the specific implementation details of the *DropBear* SSH server and client. This part might be somewhat
tedious, as it mentions many names of *structs* and functions, but it may help clarify steps from the TL;DR section
and demonstrate them through code.

**Important note on links:** I don't like blog posts containing too many links, because they give me really bad FOMOs.
Therefore, I decided *not* to link to the source code. Instead, I provide the code snippets that I find necessary, 
as well as the names of the different functions and *struct*s.

# Client Side

**Setup**

When the client reads the command line, it parses all forwarding "rules" and adds them to a list in `cli_opts.localfwds`. 

```c
void cli_getopts(int argc, char ** argv) {
    ...
    case 'L':  // for the command-line flag "-L"
        opt = OPT_LOCALTCPFWD;
        break;
    ...
    if (opt == OPT_LOCALTCPFWD) {
        addforward(&argv[i][j], cli_opts.localfwds);
    }
    ...
}
```

In the client's session loop, the function `setup_localtcp()` is called. 
This function iterates on all TCP forwarding entries that were previously added and calls `cli_localtcp` on each.

```c
void setup_localtcp() {
    ...
    for (iter = cli_opts.localfwds->first; iter; iter = iter->next) {
        /* TCPFwdEntry is the struct that holds the forwarding details -
           local and remote addresses and ports */
        struct TCPFwdEntry * fwd = (struct TCPFwdEntry*)iter->item;
        ret = cli_localtcp(
                fwd->listenaddr,
                fwd->listenport,
                fwd->connectaddr,
                fwd->connectport);
    ...
}
```


The function `cli_localtcp()` creates a `TCPListener` entry. This entry specifies not only the forwarding details, 
but also the type of the channel that should be created (*cli_chan_tcplocal*) and the TCP type (*direct*). 
For each new TCP listener, it calls `listen_tcpfwd()`.

```c
static int cli_localtcp(const char* listenaddr, 
        unsigned int listenport, 
        const char* remoteaddr,
        unsigned int remoteport) {

    struct TCPListener* tcpinfo = NULL;
    int ret;

    tcpinfo = (struct TCPListener*)m_malloc(sizeof(struct TCPListener));

    /* Assign the listening address & port, 
       and the remote address & port*/
    tcpinfo->sendaddr = m_strdup(remoteaddr);
    tcpinfo->sendport = remoteport;

    if (listenaddr)
    {
        tcpinfo->listenaddr = m_strdup(listenaddr);
    }
    else
    {
        ...
        tcpinfo->listenaddr = m_strdup("localhost");
        ...
    }
    tcpinfo->listenport = listenport;

    /* Specify channel type and TCP type */
    tcpinfo->chantype = &cli_chan_tcplocal;
    tcpinfo->tcp_type = direct;

    ret = listen_tcpfwd(tcpinfo, NULL);
    ...
}
```

**Creating a Socket**

`listen_tcpfwd()` does the following:
    
- calls `dropbear_listen()` to start listening on the local address and port.
This is where sockets are actually created and bound to the client-provided address and port.

- creates a new `Listener` object. This object contains the sockets on which listening should take place, 
and also specifies an "acceptor" - a callback function that is responsible for calling `accept()`.
Each listener is constructed with the sockets that `dropbear_listen()` returned, and with an acceptor function named 
`tcp_acceptor()`.

```c
int listen_tcpfwd(struct TCPListener* tcpinfo, struct Listener **ret_listener) {

	char portstring[NI_MAXSERV];
	int socks[DROPBEAR_MAX_SOCKS];
	int nsocks;
	struct Listener *listener;
	char* errstring = NULL;

	snprintf(portstring, sizeof(portstring), "%u", tcpinfo->listenport);
	
	/* Create sockets and listen on them */
	nsocks = dropbear_listen(tcpinfo->listenaddr, portstring, socks, 
			DROPBEAR_MAX_SOCKS, &errstring, &ses.maxfd);
	...
	/* Put the list of sockets in a new Listener object */
	listener = new_listener(socks, nsocks, CHANNEL_ID_TCPFORWARDED, tcpinfo, 
			tcp_acceptor, cleanup_tcp);
	...
}
```

**Creating a Channel**

`tcp_acceptor()` is responsible for accepting connections to a socket. **This is where the action happens**.
The function creates a new channel of type *cli_chan_tcplocal* (for local port forwarding), 
then calls `send_msg_channel_open_init()` to:
 1. inform the server of the client's newly-created channel; 
 2. tell it to create a channel of its own. 
 
Once this `CHANNEL_OPEN` message is sent successfully, the client fetches the addresses and ports from the listener object, 
and puts them inside the payload to be sent.

```c
static void tcp_acceptor(const struct Listener *listener, int sock) {

    int fd;
    struct sockaddr_storage sa;
    socklen_t len;
    char ipstring[NI_MAXHOST], portstring[NI_MAXSERV];
    struct TCPListener *tcpinfo = (struct TCPListener*)(listener->typedata);

    len = sizeof(sa);

    fd = accept(sock, (struct sockaddr*)&sa, &len);
    ...
    if (send_msg_channel_open_init(fd, tcpinfo->chantype) == DROPBEAR_SUCCESS) {
        char* addr = NULL;
        unsigned int port = 0;

        if (tcpinfo->tcp_type == direct) {
            /* "direct-tcpip" */
            /* host to connect, port to connect */
            addr = tcpinfo->sendaddr;
            port = tcpinfo->sendport;
        } 
        ...
        /* remote ip and port */
        buf_putstring(ses.writepayload, addr, strlen(addr));
        buf_putint(ses.writepayload, port);

        /* originator ip and port */
        buf_putstring(ses.writepayload, ipstring, strlen(ipstring));
        buf_putint(ses.writepayload, atol(portstring));
     ...
}
```

Whenever the server responds with its own channel ID - the client will be able to to send and receive data based on the
specified forwarding rule.

# Server Side

**Creating a Channel (upon client's request)**

The server is informed of the local port forwarding only the moment it receives the `MSG_CHANNEL_OPEN` message from the
client. Handling this message happens in the function 
`recv_msg_channel_open()`.
This function creates a new channel of type *svr_chan_tcpdirect*, and calls the channel initialization function,
named `newtcpdirect()`.

```c
void recv_msg_channel_open() {
    ...
    /* get the packet contents */
    type = buf_getstring(ses.payload, &typelen);
    
    remotechan = buf_getint(ses.payload);
    ...
    
    /* The server finds out the type of the client's channel,
       to create a channel of the same type ("direct-tcpip") */
    for (cp = &ses.chantypes[0], chantype = (*cp);  
            chantype != NULL;
            cp++, chantype = (*cp)) {
        if (strcmp(type, chantype->name) == 0) {
            break;
        }
    }
    
    /* create the channel */
    channel = newchannel(remotechan, chantype, transwindow, transmaxpacket);
    ...
    
        /* This is where newtcpdirect is called */
    if (channel->type->inithandler) {
        ret = channel->type->inithandler(channel);
        ...
    }
    ...	
    send_msg_channel_open_confirmation(channel, channel->recvwindow,
            channel->recvmaxpacket);
    ...
}
```

**Creating a Socket**

The function `newtcpdirect()` is reading the buffer `ses.payload`, which contains the data put there by the client beforehand: 
the destination host and port, and the origin host and port. 
With these details, the server connects to the remote host and port with 
`connect_remote()`.

```c
static int newtcpdirect(struct Channel * channel) {
	...
	desthost = buf_getstring(ses.payload, &len);
        ...
	destport = buf_getint(ses.payload);
	orighost = buf_getstring(ses.payload, &len);
	...
	origport = buf_getint(ses.payload);
	snprintf(portstring, sizeof(portstring), "%u", destport);
	
	/* Connect to the remote host */
	channel->conn_pending = connect_remote(desthost, portstring, channel_connect_done, channel, NULL, NULL);
	...
}
```

This function creates a connection object `c` (of type `dropbear_progress_connection`) in which it stores the remote address 
and port, a "placeholder" socket *fd* (-1) and a callback function named `channel_connect_done()`. Remember this callback for 
later on!

```c
struct dropbear_progress_connection *connect_remote(const char* remotehost, const char* remoteport,
	connect_callback cb, void* cb_data, 
	const char* bind_address, const char* bind_port)
{
	struct dropbear_progress_connection *c = NULL;
    ...
        /* Populate the connection object */
	c = m_malloc(sizeof(*c));
	c->remotehost = m_strdup(remotehost);
	c->remoteport = m_strdup(remoteport);
	c->sock = -1;
	c->cb = cb;
	c->cb_data = cb_data;

	list_append(&ses.conn_pending, c);
	...
        /* c->res contains addresses resolved from the remotehost & remoteport.
           This is a list of addresses, for each a socket is needed. */
	err = getaddrinfo(remotehost, remoteport, &hints, &c->res);
	if (err) {
		...
	} else {
		c->res_iter = c->res;
	}
	...
	return c;
}
```

As you can see in the snippet above, the connection structure is added to the list `sess.conn_pending`. This list
will be handled in the next iteration of the session's main loop.
The connection itself is done from within `connect_try_next()` function which is called by `set_connect_fds()`. 
This is where the socket is practically connected to the remote host.

```c
static void connect_try_next(struct dropbear_progress_connection *c) {
	struct addrinfo *r;
	int err;
	int res = 0;
	int fastopen = 0;
        ...
	for (r = c->res_iter; r; r = r->ai_next)
	{
		dropbear_assert(c->sock == -1);
		c->sock = socket(r->ai_family, r->ai_socktype, r->ai_protocol);
		...
		if (!fastopen) {
			res = connect(c->sock, r->ai_addr, r->ai_addrlen);
		}
                ...
	}
	...	
}
```

**Assigning the Socket to the Channel**

The status of the connection to the remote host is checked in `handle_connect_fds`, also from the session loop. 
This is where, if the connection succeeded, its callback is invoked - remember our callback? 

The callback function
`channel_connect_done()` receives the socket and the channel itself (in the *user_data* parameter). 
The function sets the socket's *fd* to be the channel's source of reading
and the destination of writing. With that, all I/O is done against the remote address.

Finally, a confirmation message is sent back to the client with the channel identifier. 

```c
void channel_connect_done(int result, int sock, void* user_data, 
                          const char* UNUSED(errstring)) {

	struct Channel *channel = user_data;

	if (result == DROPBEAR_SUCCESS)
	{
		channel->readfd = channel->writefd = sock;
		...
		send_msg_channel_open_confirmation(channel, channel->recvwindow,
				channel->recvmaxpacket);
	        ...
	}
	...
}
```

**From this point on, there's a connection between the server's channel and the client's channel. 
On the client side, the channel is reading/writing from/to the socket connected to the local address. 
On the server side, the channel reads/writes against the remote address socket.**


## Epilogue

This type of write-up is pretty difficult to write, and even more so to read. So I'm glad you survived. 

If you have any questions regarding the port forwarding mechanism - please don't hesitate to reach out and ask!
I think the process can be quite confusing (at least it was for me) - and I'd love to know which parts need more
sharpening.

It is interesting to go further and understand the session loop itself; when is reading and writing triggered?
how exactly server-side and client-side channels are paired? I figured if I went into these as well - this post
would become a complete RFC explanation ;) So if you find these topics interesting, I encourage you to read the source code 
and get the answers.

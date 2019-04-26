[[
title: Delegating Sub Domains
author: winxp5421
description: DNS sub-domain delegation.
tags: [Server-Admin]
]]

### Delegating to another domain's Name Server
Delegating a sub-domain to another name server can be done by adding a NS record under the sub domain.
If the Top Level Domain (TLD) in question is `example.com` and we wanted to delegate control over the sub domain `sub.example.com` we can simply add a new NS record under `sub.domain.com` that resolves to `ns1.OTHERdomain.com`. 
It is that simple but, only if you are delegating access to a name server that can be resolved by another domain e.g. `otherdomain.com`

If you do not have a name server that can be resolved via a separate TLD domain. Then things get a little more "sticky".

### Delegating a sub domain using a glue record

If you simply add a NS record under the sub-domain like this:

```

{
	#TLD Zone file example.com

	#DNS Servers:
	ns1.google.com 50.50.50.50
	ns2.google.com 50.50.50.50

	#Host Records:
	example.com            A 127.0.0.1
	www.example.com        A 127.0.0.1
	hotdam.example.com     A 127.0.0.1

	sub.example.com       NS ns1.sub.example.com
	sub.example.com       NS ns2.sub.example.com
}

{
	#Zone file sub.example.com
	#DNS Servers:
	ns1.sub.example.com 1.1.1.1
	ns2.sub.example.com 2.2.2.2

	#Host Records:
	something.sub.example.com A 10.10.10.10
}

```

The DNS work flow will be stuck in a "loop".

A client asks for a resolve for `somthing.sub.example.com` from example.com's DNS servers "ns[1|2].google.com"
These DNS servers will reply with please ask "ns[1|2].sub.example.com" as I do not manage that sub-domain.
The client will then ask example.com's DNS servers "ns[1|2].google.com" to resolve `ns1.sub.example.com` but again will reply with
I do not manage that sub-domain please ask "ns[1|2].sub.example.com" the client will never reach the actual name server that manages the sub-domain of
`sub.example.com` to receive a response for `somthing.sub.example.com`

We need to stop this behavior.


In order to have a sub-domain with a name server underneath the same TLD domain i.e.. ns1.sub.example.com
You must also have what is commonly referred to as a Glue record (get it sticky... glue HA!)
A Glue record is nothing more than a standard A record under the TLD of the domain.
This Glue record will be housed under the TLD domain and will resolve to the IP address of the sub-domain's name server(s).
So a full example of this environment would be:

example.com's root domain name servers are:
`ns1.google.com`
`ns2.google.com`

and you want to delegate access to `sub.domain.com` to another name server with a FQDN of `ns1.sub.example.com`
```
{
	TLD Zone file example.com

	#DNS Servers:
	ns1.google.com 50.50.50.50
	ns2.google.com 50.50.50.50

	#Host Records:
	example.com            A 127.0.0.1
	www.example.com        A 127.0.0.1
	hotdam.example.com     A 127.0.0.1
	ns1.sub.example.com    A 1.1.1.1 (Glue Record)
	ns2.sub.example.com    A 2.2.2.2 (Glue Record)

	sub.example.com       NS ns1.sub.example.com
	sub.example.com       NS ns2.sub.example.com
}

{
	#Zone file sub.example.com
	#DNS Servers:
	ns1.sub.example.com 1.1.1.1
	ns2.sub.example.com 2.2.2.2

	#Host Records:
	something.sub.example.com A 10.10.10.10
}
```
Now the name servers `1.1.1.1 (ns1.subdoman.com)` and `2.2.2.2 (ns2.sub.example.com)` have full control over the `sub.example.com` Zone
These will act independently from the main TLD Name server that serves the `example.com` domain.

How does this work exactly?

A client asks for a resolve for `somthing.sub.example.com` from example.com's DNS servers "ns[1,2].google.com"
These DNS servers will reply with please ask "ns[1,2].sub.example.com" as I do not manage that sub-domain.
The client will then ask example.com's DNS servers "ns[1,2].google.com" to resolve `ns1.sub.example.com` which will return an ip of `1.1.1.1`
The client will now ask `1.1.1.1 (ns1.sub.example.com)` for a resolve for `something.sub.example.com` and `1.1.1.1` will replay with the correct IP of `10.10.10.10`
and the connection can be completed.

# Recursive DNS servers

Recursive DNS servers receive DNS requests and forward the requests to the appropriate servers, returning the results to the client.

Authoritative DNS servers receive DNS requests only for domains for which they are ultimately responsible.

Running your own recursive DNS servers removes your reliance on your ISP provided recursive DNS servers. It also reduces the latency between your clients and the DNS server by moving it in house. As an added benefit you can be sure that the queries are not being filtered by your DNS provider.

## Choice of software

### PowerDNS Recursor

PowerDNS is available in the Ubuntu repositories in 18.04, 20.04, and 22.04. Running on 22.04 is required to get the newest version.

Install with

    sudo apt install pdns-recursor

Configure with the bare minimum configuration

    local-address=ip to listen on

If you have any internal domains that this resolver should look to a specific server (or VIP) for, add a forward-zones line.

    forward-zones=corp.yourdomain.lan=192.0.2.101, legacy.internal=192.0.2.101

If you want to enable the webserver, you must configure the listen address, allow from, and API key.

    api-key=secure key here
    webserver=yes
    webserver-address=127.0.0.1
    webserver-allow-from=127.0.0.1,::1

Note that in this configuration you can only access the webserver from localhost. You can use a reverse proxy webserver such as nginx to restrict access to this page.

### Knot Resolver

Install with

    wget https://secure.nic.cz/files/knot-resolver/knot-resolver-release.deb
    sudo dpkg -i knot-resolver-release.deb
    sudo apt update
    sudo apt install -y knot-resolver

Configure with the bare minimum configuration

    --net.listen('ip to listen on', 53, { kind = 'dns' })

If you have any internal domains that this resolver should look to a specific server (or VIP) for, add an internalDomains line

    internalDomains = policy.todnames({'corp.yourdomain.lan', 'legacy.internal'})

If you wish to disable cache for these domains, add

    policy.add(policy.suffix(policy.FLAGS({'NO_CACHE'}), internalDomains))

Then finally configure the internal server to query for these domains.

    policy.add(policy.suffix(policy.STUB({'192.0.2.101'}), internalDomains))

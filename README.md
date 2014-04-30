#SkyDNS [![Build Status](https://travis-ci.org/skynetservices/skydns.png)](https://travis-ci.org/skynetservices/skydns2)
*Version 2.0.0*

SkyDNS2 is a distributed service for announcement and discovery of services build on
top of [etcd](https://github.com/coreos/etcd). It utilizes DNS queries
to discover available services. This is done by leveraging SRV records in DNS,
with special meaning given to subdomains, priorities and weights.

This is the original [announcement blog post](http://blog.gopheracademy.com/skydns) for version 1, 
since then SkyDNS has seen some changes, most notably to ability to use etcd as a backend.

##Setup / Install
Compile/download and run etcd, see the documentation for etcd at <https://github.com/coreos/etcd>.

Then compile SkyDNS, and execute it.

`go get -d -v ./... && go build -v ./...`

SkyDNS' configuration is stored *in* etcd, there are no flags. To start SkyDNS set the
etcd machines in the variable ETCD_MACHINES:

    export ETCD_MACHINES='http://127.0.0.1:4001'
    ./skydns2


## Configuration
SkyDNS' configuration is stored inside `etcd`, under the key `/skydns/config`, the following paramaters
may be set:

* `dns_addr`: ip:port on which the SkyDNS should start the DNS server, defaults to `127.0.0.1:53`.
* `domain`: domain SkyDNS is authoritative for, default to `skydns.local.`.
* `dnssec`: enable DNSSEC.
* `round_robin`: enable round robin sorting for A and AAAA responses, defaults to true
* `nameservers`: forward DNS requests to these nameservers (ip:port combination), when we are not
    authoritative for them.
* `read_timeout`: network read timeout, for DNS and talking with etcd.
* `write_timeout`: network write timeout.
* `ttl`: default TTL in seconds to use on replies when none is set in etcd, defaults to 3600.
* `min_ttl`: minimum TTL in seconds to use on NXDOMAIN, defaults to 30.

To set the configuration so something like:

    curl -XPUT http://127.0.0.1:4001/v2/keys/skydns/config \
    -d value='{"dns_addr":"127.0.0.1:5354","ttl":3600}'

### Service Announcements
You announce your service by submitting JSON over HTTP to etcd with information about your service.
This information will then be available for queries via DNS.

When providing information you will need to fill out the following values.

* Path - The path of the key in etcd, e.g. if the domain you want to register is "rails.production.east.skydns.local", you need to reverse
    it and replace the dots with slashes. So the name here becomes: local/skydns/east/production/rails. Then prefix the `/skydns/` string to,
    so the final path becomes `/v2/keys/skdydns/local/skydns/east/production/rails`
* Host - The name of your service, e.g., "service5.mydomain.com" or and ip address (either v4 or v6)
* Port - the port where the service can be reached.
* Priority - the priority of the service.

Adding the service can thus be done with:

    curl -XPUT http://127.0.0.1:4001/v2/keys/local/skydns/east/production/rails \
    -d value='{"host":"service5.example.com","priority":20}'

When querying the DNS for services you can use subdomains. see the section named "Subdomains" below for more information.

### Service Discovery via the DNS

##Discovery (DNS)
You can find services by querying SkyDNS via any DNS client or utility. It uses a known domain syntax with subdomains to find matching services.

For the purpose of this document, lets suppose we have added to following services to etcd:

* 1.rails.production.east.skydns.local, mapping to service1.example.com
* 2.rails.production.west.skydns.local, mapping to service2.example.com
* 4.rails.staging.east.skydns.local, mapping to 10.0.1.125
* 6.rails.staging.east.skydns.local, mapping to 2003::8:1

These names can be added with:

    curl -XPUT http://127.0.0.1:4001/v2/keys/local/skydns/east/production/rails/1 \
    -d value='{"host":"service1.example.com"}'
    curl -XPUT http://127.0.0.1:4001/v2/keys/local/skydns/west/production/rails/2 \
    -d value='{"host":"service2.example.com"}'
    curl -XPUT http://127.0.0.1:4001/v2/keys/local/skydns/east/staging/rails/4 \
    -d value='{"host":"10.0.1.125"}'
    curl -XPUT http://127.0.0.1:4001/v2/keys/local/skydns/east/staging/rails/6 \
    -d value='{"host":"2003::8:1"}'

Testing one of the names with `dig`

    % dig @localhost SRV 1.rails.production.east.skydns.local
    ;; QUESTION SECTION:
    ;1.rails.production.east.skydns.local.	IN	SRV

    ;; ANSWER SECTION:
    1.rails.production.east.skydns.local.   3600	IN	SRV	10 0 9000   service1.example.com.

#### Subdomains

Of course using the full names isn't *that* useful, so SkyDNS lets you query for subdomains, and returns responses based upon the amount of services matched
by the subdomain.

If we are interesting in all the servers in the east-region, we can just leave of the right most labels from our query:

    % dig @localhost SRV east.skydns.local
    ;; QUESTION SECTION
    ; east.skydns.local.    IN      SRV

    ;; ANSWER SECTION
    east.skydns.local.

This returns all services which has their suffix matching `east.skydns.local`, the more labels you add the more specific your search becomes.

Using wildcards `*` in the middle of the query (as could be done in SkyDNS version 1), is not supported anymore.

###Examples

Now we can try some of our example DNS lookups:

#####All Services in Production 
`dig @localhost production.skydns.local SRV`

	;; QUESTION SECTION:
	;production.skydns.local.			IN	SRV

	;; ANSWER SECTION:
	production.skydns.local.		629	IN	SRV	10 20 80   web1.site.com.
	production.skydns.local.		3979	IN	SRV	10 20 8080 web2.site.com.
	production.skydns.local.		3629	IN	SRV	10 20 9000 server24.
	production.skydns.local.		3985	IN	SRV	10 20 80   web3.site.com.
	production.skydns.local.		3990	IN	SRV	10 20 80   web4.site.com.

####A/AAAA Records
To return A records, simply run a normal DNS query for a service matching the above patterns.

Now do a normal DNS query:
`dig rails.production.skydns.local`

	;; QUESTION SECTION:
	;rails.production.skydns.local.	IN	A

	;; ANSWER SECTION:
	rails.production.skydns.local. 399918 IN A	127.0.0.10
	rails.production.skydns.local. 399918 IN A	127.0.0.11
	rails.production.skydns.local. 399918 IN A	127.0.0.12
	rails.production.skydns.local. 399919 IN A	127.0.0.13

Now you have a list of all known IP Addresses registered running the `rails`
service name. Because we're returning A records and not SRV records, there
are no ports listed, so this is only useful when you're querying for services
running on ports known to you in advance. Notice, we didn't specify version or
region, but we could have.

####DNS Forwarding

By specifying `-nameserver="8.8.8.8:53,8.8.4.4:53` on the `skydns` command line,
you create a DNS forwarding proxy. In this case it round robins between the two
nameserver IPs mentioned on the command line.

Requests for which SkyDNS isn't authoritative
will be forwarded and proxied back to the client. This means that you can set
SkyDNS as the primary DNS server in `/etc/resolv.conf` and use it for both service
discovery and normal DNS operations.

*Please test this before relying on it in production, as there may be edge cases that don't work as planned.*

####DNSSEC

SkyDNS support signing DNS answers (also know as DNSSEC). To use it you need to
create a DNSSEC keypair and use that in SkyDNS. For instance if the domain for
SkyDNS is `skydns.local`:

    dnssec-keygen skydns.local
    Generating key pair............++++++ ...................................++++++
    Kskydns.local.+005+49860

This creates two files both with the basename `Kskydns.local.+005.49860`, one of the
extension `.key` (this holds the public key) and one with the extension `.private` which
hold the private key. The basename of this file should be given to SkyDNS's -dnssec
option: `-dnssec=Kskydns.local.+005+49860`

If you then query with `dig +dnssec` you will get signatures, keys and NSEC3 records returned.

## License
The MIT License (MIT)

Copyright © 2014 The SkyDNS Authors

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
THE SOFTWARE.

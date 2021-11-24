# Handshake Query

⚠️ Usage of this library is not currently recommended in your application as the API will likely change.

Handshake Query is a cross-platform library to trustlessly resolve and verify Handshake names using an SPV node. Supports DNSSEC & DNS-Based Authentication of Named Entities (DANE). It wraps [libhsk](https://github.com/handshake-org/hnsd) with a thread-safe API. It's currently being used by Impervious browser.
## Supported Platforms

iOS, Android, macOS, Windows and Linux

## Usage

### Launching an SPV node

This example shows how to launch an SPV node, wait for it to sync and store block headers in a temp directory.

```go
package main

import (
	hns "github.com/imperviousinc/hnsquery"
)

config := &hns.Config {
    // Used for storing cache data such as block headers 
    DataDir: os.TempDir(),
}


client, err := hns.NewClient(config)
if err != nil { ... }
defer client.Destroy()

ready := make(chan error)
client.Start(ready)

<-ready // blocks until SPV node is synced

// Get proofofconcept zone
zone, err := client.GetZone("proofofconcept")
for _, rr := range zone {
   fmt.Println(rr)
}

// Read info
fmt.Println("Height: ", client.Height())
fmt.Println("Sync progress: ", client.Progress())
fmt.Println("Peers: ", client.PeerCount())
fmt.Println("Active Peers:", client.ActivePeerCount())
```

### Resolving names

```go
// create a Proof of work trust anchor using the client
powTA := func(ctx context.Context, cut string) (*dnssec.Zone, bool, error) {
	// Follow example in mobile package
}

// initialize a resolver in forwarding mode with DoH
resolver, err := hns.NewResolver(&ResolverConfig{
        TrustAnchorFunc: powTA,
	Forward: "https://hs.dnssec.dev/dns-query"
})

// Securely resolve names with trustless DNSSEC validation
resolver.Query("_443._tcp.proofofconcept.", dns.TypeTLSA)

```


### Verifying certificates
You can create custom cert verifiers but in most cases you may want to use the default:
```go
cv := hns.NewDNSCertVerifier(resolver)
cv.Verify(ctx, &CertVerifyInfo{
    Host: "proofofconcept",
    Port: "443",
    Protocol: "tcp",
    RawCerts: certs
})
```

## DNSSEC validation

Handshake Query provides a modern Handshake native DNSSEC validation package that doesn't rely on a root KSK. Although this is optional as it can be integrated with other libraries such as libunbound to support a recursive mode (TODO)

[RFC8624](https://datatracker.ietf.org/doc/html/rfc8624) still considers weak crypto such as 256-bit RSA key size to be secure. The web has moved on. hnsq will downgrade algorithms it considers weak and they cannot be used for DANE. The following table shows which algorithms are accepted: 
```
+--------+--------------------+----------------------------------+
| Number | Mnemonics          | Supported for DANE               |
+--------+--------------------+ ---------------------------------+
| 1      | RSAMD5             | NO                               |
| 3      | DSA                | NO                               |
| 5      | RSASHA1            | NO                               |
| 6      | DSA-NSEC3-SHA1     | NO                               |
| 7      | RSASHA1-NSEC3-SHA1 | NO                               |
| 8      | RSASHA256          | YES - Min key size 2048 bit      |
| 10     | RSASHA512          | YES - Min key size 2048 bit      |
| 12     | ECC-GOST           | NO                               |
| 13     | ECDSAP256SHA256    | YES                              |
| 14     | ECDSAP384SHA384    | YES                              |
| 15     | ED25519            | YES                              |
| 16     | ED448              | TODO                             |
+--------+--------------------+----------------------------------+
```

### PoWDoH

PoWDoH requests the DNSSEC chain from a DoH server and verifies it by fetching the DS record from an SPV node which verifies Proof of Work.

DNS records & DNSSEC signatures can be transmitted over any channel. DoH transmits the signatures over HTTPS. The advantage is better censorship resistance compared to using port 53 since queries hide with other HTTPS traffic. Plain DNS traffic is unreliable due to middlebox inteference and they are easy to block by ISPs.

Using a forwarding resolver is faster than doing recursion since it benefits from a global cache. Currently, hnsq queries DNS records over DoH and re-uses tcp connections to reduce latency but performance can be improved with CHAIN queries (RFC7901).

PoWDoH may improve security if the DoH server uses cert pinning & also performs DNSSEC validation because an attacker needs to compromise the DoH server (which likely uses a full node) in addition to the SPV node.

### TLS DNSSEC Chain Extension (RFC9102)

The DNSSEC chain extension is an experimental TLS extension that embeds the DNSSEC chain which obviates the need to perform separate, out-of-band DNS lookups. The complete chain can be validated directly with an SPV node. No need for an external forwarding or recursive resolver.

Not currently supported by either clients or servers.

TODO.


## Build

Note: these instructions are not yet complete but you should be able to build it if you're familar with cgo.

### iOS


```
$ git clone https://github.com/buffrr/hnsd && cd hnsd
$ git checkout ios
$ ./build-ios.sh
$ gomobile bind -target ios/arm64 -o MobileHNS.xcframework github.com/imperviousinc/hnsquery/mobile
```

### Android

You can build it with gomobile. You also need NDK to compile libhsk.

TODO


### MacOS, Linux and Windows

build libhsk & hnsq
```
$ ./configure --without-daemon --prefix /path/to/build/dir
$ make -j 10
$ make install
$ go build
```







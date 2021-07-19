## Introduction

DNSLink is the specification of a format for [DNS][] [`TXT` records][TXT] that allows the association of [decentralized web][dweb] (dweb) resources with a [domain][].

![](./img/dns-query.png)

## How does it work?

When you register a domain at your name registrar (eg. namecheap.com) you can set `TXT` entries additionally to `A` or `CNAME`.

DNSLink specifies a format for `TXT` entries that allow you to _link_ dweb resources to that domain.

Using just a DNS client, anyone can quickly lookup the link and request the resource through a decentralized network.

Here is an example for a valid DNSLink `TXT` entry:

```
dnslink=/ipfs/QmaCveU7uyVKUSczmiCc85N7jtdZuBoKMrmcHbnnP26DCx
```

### Tools

You _can_ use `dig` to lookup the `dnslink=` TXT entries for at domain. <br>_(for Windows users: `dig.exe` is part of [the ISC's `bind` package](https://www.isc.org/download/))_

```sh
> dig +short TXT _dnslink.dnslink.dev
_dnslink.dnslink-website.on.fleek.co.
"dnslink=/ipfs/QmaCveU7uyVKUSczmiCc85N7jtdZuBoKMrmcHbnnP26DCx"
```

Considering the subtle details of DNSLink however, it is better to use the `dnslink` CLI client provided with either [`@dnslink/js`][js] or [`dnslink-std/go`][go].

```sh
> dnslink dnslink.dev
/ipfs/QmaCveU7uyVKUSczmiCc85N7jtdZuBoKMrmcHbnnP26DCx    [ttl=120]
```


### Format

DNSLink entries are of the form:

```
dnslink=/<key>/<value>
```

- The prefix `dnslink=/` is there to signal that this `TXT` record value is a DNSLink. This is important because `TXT` records are used for many purposes that DNSLink should not interfere with.
- As there are more than one decentralized system, the `<key>` indicates which decentralized system you want to use.
- The `<value>` is the link you want to set. The `<value>` is any link you wish to use. It could be a URL or a path.

### `_dnslink` subdomain

DNSLink entries can be specified directly on the domain or on the `_dnslink` subdomain. The entries specified in the **`_dnslink` subdomain are given priority!**

For example, lets say you specify two entries like this:

```shell
> dig +short TXT fictive.domain
dnslink=/ipfs/QmAAAA....AAAA

> dig +short TXT _dnslink.fictive.domain
dnslink=/ipfs/QmBBBB....BBBB
```

Looking up `fictive.domain` will return the entry of `_dnslink.fictive.domain`!

```shell
> dnslink fictive.domain
/ipfs/QmBBBB....BBBB [ttl=60]
```

Using a `_dnslink` subdomain prevents conflicts with `CNAME` and `ALIAS` records. It is also allows you to give control over your DNSLink records to a third party without giving away full control over the original DNS zone. 

`TXT` records on the `domain` without the `_dnslink` subdomain are discouraged and exist for legacy reasons. We historically started with supporting records on the normal domain, and only found the issue with `CNAME` and `ALIAS` records later on.

It is _not required_ but strongly recommended to **set the dnslink entry on the `_dnslink` subdomain!**

### Multiple entries

#### Multiple `dnslink=` `TXT` entries per domain

There is more than one kind of dweb resource out there. [`ipfs`][ipfs] and [`hyper`][hyper] as just two popular examples. In order for domains to be able to provide multiple representations, you can specify more than one `dnslink=/<key>/<value>` TXT entry per domain. DNSLink compatible implementations are encouraged to return a result that groups results by `key`.

```js
const { links } = await resolveN('dnslink.dev')
links.ipfs //...
links.hyper //...
```

#### Multiple values per key

Most dweb formats have a 1:1 relationship. For example, one domain is supposed to point to one `ipfs` resource. But some systems - like [multiaddr][] - may make use of multiple links to link more than one well-known peer. Other systems may add meta information. For that resason DNSLink clients need to return a list of links per `key`.

```js
Array.isArray(links.ipfs) // true if ipfs links are given.
```

#### Sorting

DNSLink values need to be sorted alphanumerically by lower entries first. This is relevant because conventional DNS clients - like `dig` - do not need to return TXT entries in a sorted manner!

```sh
# OK
dnslink=/foo/B
dnslink=/foo/C
dnslink=/bar/A # Its okay for the key to be unsorted!

# NOT OK
dnslink=/foo/C # ... but the value needs to be sorted!
dnslink=/foo/B
dnslink=/bar/A
```

### Chaining and redirects

Additionally to the regular `CNAME` method for redirects, DNSLink supports redirects using the special `dnslink` key.

The redirect can be of the format:

```
dnslink=/dnslink/<redirect>
```

Here is an example for how a redirect works:

```sh
> dig +short TXT _dnslink.t09.dnslink.dev
dnslink=/dnslink/b.t09.dnslink.dev

> dig +short TXT _dnslink.b.t09.dnslink.dev
dnslink=/ipfs/AADE

> dnslink t09.dnslink.dev
/ipfs/AADE      [ttl=3600]
```

#### Precendence

Redirects take priority over other `dnslink=` entries! If a redirect is present the other entries will be ignored!

```sh
> dig +short TXT t14.dnslink.dev
"dnslink=/dnslink/1.t14.dnslink.dev"
"dnslink=/ipfs/mnop"

> dig +short TXT 1.t14.dnslink.dev
"dnslink=/ipns/AALM"

> dnslink t14.dnslink.dev
/ipns/AALM      [ttl=3600]
```

#### Limits to Redirects

DNSLink implementations need to limit redirects to **at most 32**. Counting even fallbacks from `_dnslink.<domain>` to `<domain>` as redirect. Redirects exceeding that number _(or are identified as circluar)_ are treated as errors.


The number of redirects needs to be consistent between all implementations for a predictable client behavior.

Redirects need to point to valid dns domains. This means that they can have at most `253` characters in total with `63` characters max per subdomain. No domain part may be empty.

```sh
# invalid: empty tld
dnslink=/dnslink/.
# invalid because of empty subdomain
dnslink=/dnslink/foo..bar
# too long subdomain (exceeds 63 chars)
dnslink=/dnslink/abc{ +54 chars }efg.com
# too long (exceeds 253 chars)
dnslink=/dnslink/_dnslink.abc{ +239 chars }efg
# too long because _dnslink. prefix is added
dnslink=/dnslink/abc{ +239 chars }efg
```

#### Paths

When specifying a redirect you may also specify a deep link on the redirect:

```
dnslink=/dnslink/target.domain/some/path?foo=bar
```

In this example **`/some/path?foo=bar`** is a deep link. DNSLink does not specify _how_ these paths are used by the different dweb formats. `ifps` may use these differently from `hyper`. You need to lookup the respective specification or implementation to see how paths are used.

The DNSLink implementations will return any given path specification as separate `path` list that needs to be combined by the user:

```js
const { links, path } = await resolveN('redirecting.domain')

path[0]
// {
//   pathname: '/some/path',
//   search: {
//     foo: ['bar']
//   }
// }
```

The standard libraries are encouraged to provide a `reducePath` function that can be used to combine/reduce a given path.

```js
const { links, path } = await resolveN('redirecting.domain')
reducePath(links.ipfs[0], path) // <value>/some/path?foo=bar
```

### Encoding

DNSLink entries are limited to [ASCII][] characters. For Unicode/UTF-8 characters, DNSLink clients support [percent encoding][%-encoding] transparently.

```shell
# Invalid
dnslink=/aフ/bar
dnslink=/foo/bホ

# Valid
dnslink=/%E3%83%95%E3%82%B2/bar
dnslink=/foo/%E3%83%9B%E3%82%AC
```

The valid entries will be decoded by the libraries:

```javascript
const { links } = await resolveN('...')

deepEqual(links, {
  'フゲ': ['bar'],
  'foo': ['ホガ']
})
```

_Note:_ Technically `TXT` entries are transported as binary data and could be interpreted as UTF-8 characters. As the encoding outside the ASCII
space is not specified DNSLink does not allow these, to prevent compilations with other DNS tools.

[DNS]: https://en.wikipedia.org/wiki/Domain_Name_System
[TXT]: https://en.wikipedia.org/wiki/TXT_record
[dweb]: https://en.wikipedia.org/wiki/Decentralized_web
[domain]: https://en.wikipedia.org/wiki/Domain_name
[js]: https://npmjs.com/package/@dnslink/js
[ASCII]: https://en.wikipedia.org/wiki/ASCII
[go]: https://github.com/dnslink-std/go/releases
[js-cli]: https://github.com/dnslink-std/js#command-line
[go-cli]: https://github.com/dnslink-std/go#command-line
[ipfs]: https://ipfs.io
[hyper]: https://hypercore-protocol.org/protocol/#hyperdrive
[multiaddr]: https://multiformats.io/multiaddr/
[punycode]: https://en.wikipedia.org/wiki/Punycode
[%-encoding]: https://en.wikipedia.org/wiki/Percent-encoding

## Tutorial

### Step 0: Find something to link to.

In this tutorial, we'll link to [the libp2p website](https://libp2p.io), on [IPFS](https://ipfs.io). At the time of writing this, the website had the IPFS address:

```
/ipfs/Qmc2o4ZNtbinEmRF9UGouBYTuiHbtCSShMFRbBY5ZiZDmU
```

You can view this on the global IPFS gateway: https://ipfs.io/ipfs/Qmc2o4ZNtbinEmRF9UGouBYTuiHbtCSShMFRbBY5ZiZDmU

And even download it with ipfs:

```
ipfs get /ipfs/Qmc2o4ZNtbinEmRF9UGouBYTuiHbtCSShMFRbBY5ZiZDmU
```

### Step 1: Choose a domain name to link from.

For this tutorial, I'll use the domain name `libp2p.io`. You can use whatever domain you control. I happen to be using an `ALIAS` record on this domain as well (to make the website load directly via the IPFS gateway), but that is not a problem: the convention to prefix the domain with `_dnslink.` allows for complex DNS setups like this.

So the full domain name I'll set the record on is: `_dnslink.libp2p.io`.

### Step 2: Set the DNSLink value on a `TXT` record.

Let's set the link by creating a `TXT` record on the domain name. This is going to be specific to your DNS tooling. Unfortunately there is no standard cli tool or web service to set domain names :(. Every registrar and name server has its own web app, and API tooling. (yes, this is widely regarded as a poor state of affairs).

Consider setting the record, via an imaginary dns cli tool:
```
> my-dns-tool set --type=TXT --ttl=60 --domain=libp2p.io --name=_dnslink --value="dnslink=/ipfs/Qmc2o4ZNtbinEmRF9UGouBYTuiHbtCSShMFRbBY5ZiZDmU"
_dnslink.libp2p.io TXT 60 dnslink=/ipfs/Qmc2o4ZNtbinEmRF9UGouBYTuiHbtCSShMFRbBY5ZiZDmU
```

Or directly in a Digital Ocean prompt:

![](./img/digitalocean.png)


### Step 3: Resolve the link

Now, let's try resolving the link!

You can get the link value manually using `dig` or another dns lookup tool:

```
> dig +short TXT _dnslink.libp2p.io
dnslink=/ipfs/Qmc2o4ZNtbinEmRF9UGouBYTuiHbtCSShMFRbBY5ZiZDmU
```

Extract the value with `sed`:
```
> dig +short TXT _dnslink.libp2p.io | sed -E 's/"dnslink=(.*)"/\1/g'
/ipfs/Qmc2o4ZNtbinEmRF9UGouBYTuiHbtCSShMFRbBY5ZiZDmU
```

Then, you can feed the output of that to ipfs:

```
> ipfs ls /ipfs/Qmc2o4ZNtbinEmRF9UGouBYTuiHbtCSShMFRbBY5ZiZDmU
QmeotoX2bE7fVgMvUS9ZXL2RMoZQHiuQVZZR1Hts3JVmUT 265   _previous-versions
QmP5JpytsFEQ5Y8oQ875kPrdA1dAtyXAH6U8eKbTUbptNd -     bundles/
QmcRUrMyFePBNnvoqZB5Uk7y6aoWGoUW6EB8JWUxztKTma -     categories/
QmeVM9YZStiFcjjQYpzJ1KWJsaFGcRWaeMAymSfLydu9mh -     css/
QmRJjyE1Bi64AD7MSpt4xKHaiWXq7WGAne8KTzn7UyYeWq -     fonts/
Qma6Eg1JMAPjBg827ywDG1nx4TBwxYWxQxeb1CXUkDnrHk -     img/
QmdB9xXJHNXnaiikCXVpomHriNGXwvSUqdkC1krtFq4WWW -     implementations/
QmXCq4KMZC4mdxovpnrHU9K92LVBLSExLEsrvwTGNEswnv 62880 index.html
QmQNjTorGWRTqEwctqmdBfuBBRTj8vQD3iGjNNCu7vA5i9 3138  index.xml
QmPsosZeKZTUcBkcucPtPnk3fF4ia4vBdJ6str9CRr6VTQ -     js/
QmYBUY8Y8uXEAPJSvMTdpfGoL8bujNo4RKoxkCnnKXoTD9 -     media/
QmUZ23DEtL3aUFaLgCEQv5yEDigGP2ajioXPVZZ6S7DYVa 561   sitemap.xml
QmRgig5gnP8XJ16PWJW8qdjvayY7kQHaJTPfYWPSe2BAyN -     tags/
```

Or chain it all together:

```
> dig +short TXT _dnslink.libp2p.io | sed -E 's/"dnslink=(.*)"/\1/g' | ipfs ls
QmeotoX2bE7fVgMvUS9ZXL2RMoZQHiuQVZZR1Hts3JVmUT 265   _previous-versions
QmP5JpytsFEQ5Y8oQ875kPrdA1dAtyXAH6U8eKbTUbptNd -     bundles/
QmcRUrMyFePBNnvoqZB5Uk7y6aoWGoUW6EB8JWUxztKTma -     categories/
QmeVM9YZStiFcjjQYpzJ1KWJsaFGcRWaeMAymSfLydu9mh -     css/
QmRJjyE1Bi64AD7MSpt4xKHaiWXq7WGAne8KTzn7UyYeWq -     fonts/
Qma6Eg1JMAPjBg827ywDG1nx4TBwxYWxQxeb1CXUkDnrHk -     img/
QmdB9xXJHNXnaiikCXVpomHriNGXwvSUqdkC1krtFq4WWW -     implementations/
QmXCq4KMZC4mdxovpnrHU9K92LVBLSExLEsrvwTGNEswnv 62880 index.html
QmQNjTorGWRTqEwctqmdBfuBBRTj8vQD3iGjNNCu7vA5i9 3138  index.xml
QmPsosZeKZTUcBkcucPtPnk3fF4ia4vBdJ6str9CRr6VTQ -     js/
QmYBUY8Y8uXEAPJSvMTdpfGoL8bujNo4RKoxkCnnKXoTD9 -     media/
QmUZ23DEtL3aUFaLgCEQv5yEDigGP2ajioXPVZZ6S7DYVa 561   sitemap.xml
QmRgig5gnP8XJ16PWJW8qdjvayY7kQHaJTPfYWPSe2BAyN -     tags/
```

## Usage Examples

### Example: User-friendly name resolution within IPFS

IPFS has DNSLink resolution built in though, so you could just do the following to achieve the same result as in the tutorial above.

Given this dnslink:
```
> dig +short TXT _dnslink.libp2p.io
dnslink=/ipfs/Qmc2o4ZNtbinEmRF9UGouBYTuiHbtCSShMFRbBY5ZiZDmU
```

ipfs uses DNSLink to resolve it natively:
```
> ipfs ls /ipns/libp2p.io
QmeotoX2bE7fVgMvUS9ZXL2RMoZQHiuQVZZR1Hts3JVmUT 265   _previous-versions
QmP5JpytsFEQ5Y8oQ875kPrdA1dAtyXAH6U8eKbTUbptNd -     bundles/
QmcRUrMyFePBNnvoqZB5Uk7y6aoWGoUW6EB8JWUxztKTma -     categories/
QmeVM9YZStiFcjjQYpzJ1KWJsaFGcRWaeMAymSfLydu9mh -     css/
QmRJjyE1Bi64AD7MSpt4xKHaiWXq7WGAne8KTzn7UyYeWq -     fonts/
Qma6Eg1JMAPjBg827ywDG1nx4TBwxYWxQxeb1CXUkDnrHk -     img/
QmdB9xXJHNXnaiikCXVpomHriNGXwvSUqdkC1krtFq4WWW -     implementations/
QmXCq4KMZC4mdxovpnrHU9K92LVBLSExLEsrvwTGNEswnv 62880 index.html
QmQNjTorGWRTqEwctqmdBfuBBRTj8vQD3iGjNNCu7vA5i9 3138  index.xml
QmPsosZeKZTUcBkcucPtPnk3fF4ia4vBdJ6str9CRr6VTQ -     js/
QmYBUY8Y8uXEAPJSvMTdpfGoL8bujNo4RKoxkCnnKXoTD9 -     media/
QmUZ23DEtL3aUFaLgCEQv5yEDigGP2ajioXPVZZ6S7DYVa 561   sitemap.xml
QmRgig5gnP8XJ16PWJW8qdjvayY7kQHaJTPfYWPSe2BAyN -     tags/
```

You can find out more at the [IPFS DNSLink documentation](https://docs.ipfs.io/concepts/dnslink/).

### Example: IPFS Gateway

You can also just check it out on the web. The IPFS gateway resolves DNSLink automatically too. Check it out at https://ipfs.io/ipns/libp2p.io or https://dweb.link/ipns/libp2p.io which will provide proper origin isolation for use in browsers.

**How does that work?** The gateway takes the part after `/ipns/`, if it is a DNS domain name, it checks for a DNSLink at either the domain name, or `_dnslink.` prefixed version. In this case it finds our DNSLink at `_dnslink.libp2p.io` and resolves that.

**But what about [https://libp2p.io](https://libp2p.io)?** Yes, https://libp2p.io also works, that uses a combination of DNSLink, a `ALIAS` record in `libp2p.io`, and the ipfs gateway. Basically:
1. the browser first checks for `A` records for `libp2p.io`. dns finds an `ALIAS` to `gateway-int.ipfs.io`, and those `A` records:
    ```
    > dig A libp2p.io
    gateway.ipfs.io.        119     IN      A       209.94.90.1
    ```
2. the browser then connects to `http://209.94.90.1`, using a `HOST: libp2p.io` HTTP header.
3. the ipfs gateway reads the `HOST: libp2p.io` header, and -- recognizing a DNS name -- checks for an associated DNSLink at either `libp2p.io` or `_dnslink.libp2p.io`.
    ```
    > dig +short TXT _dnslink.libp2p.io
    "dnslink=/ipfs/Qmc2o4ZNtbinEmRF9UGouBYTuiHbtCSShMFRbBY5ZiZDmU"
    ```
4. The gateway finds the link at `_dnslink.libp2p.io` leading to `/ipfs/Qmc2o4ZNtbinEmRF9UGouBYTuiHbtCSShMFRbBY5ZiZDmU`.
5. The gateway fetches the IPFS web content at `/ipfs/Qmc2o4ZNtbinEmRF9UGouBYTuiHbtCSShMFRbBY5ZiZDmU` and serves it to the browser.
6. The browser renders it happily, preserving the original pretty name of `https://libp2p.io`


### Example: IPFS Companion

Similar to the IPFS Gateway, [IPFS Companion](https://github.com/ipfs-shipyard/ipfs-companion) uses DNSLink natively within the browser to resolve IPFS web content. IPFS Companion has a feature that tests domain names for the presence of dnslink `TXT` records, and if it finds them, then it serves content via IPFS instead.

You can find out more about how it works at [IPFS Companion's DNSLink documentation](https://github.com/ipfs-shipyard/ipfs-companion/blob/master/docs/dnslink.md).

## Tools

There are a number of tools related to DNSLink.

#### go-ipfs

go-ipfs has built-in support for resolving DNSLinks:

```
$ ipfs name resolve -r /ipns/libp2p.io
/ipfs/Qmc2o4ZNtbinEmRF9UGouBYTuiHbtCSShMFRbBY5ZiZDmU
```

One can also run HTTP Gateway capable of hosting DNSLink websites. See [examples](https://github.com/ipfs/go-ipfs/blob/master/docs/config.md#gateway-recipes).

#### go-dnslink

https://github.com/ipfs/go-dnslink - dnslink resolution in go (used by [go-ipfs](https://github.com/ipfs/go-ipfs))

#### js-dnslink

https://www.npmjs.com/package/dnslink - dnslink resolution in javascript

#### dnslink-deploy

https://github.com/ipfs/dnslink-deploy - `dnslink-deploy` a tool for setting DNSLinks on Digital Ocean (and maybe more someday others):

#### ipfs-deploy

https://github.com/ipfs-shipyard/ipfs-deploy - upload static website to IPFS pinning services and optionally update DNS (Cloudflare, DNSSimple)

## Docs

- IPFS and DNSLink: https://docs.ipfs.io/concepts/dnslink/
- IPFS Companion and DNSLink: https://docs.ipfs.io/how-to/dnslink-companion/
- DNSLink in Cloudflare's Gateway: https://developers.cloudflare.com/distributed-web/ipfs-gateway/connecting-website/
- Explanation of how DNSLink and the IPFS Gateway works: https://www.youtube.com/watch?v=YxKZFeDvcBs

## Known Users

### Systems

#### IPFS

IPFS is a heavy user of DNSLink. It is used in the core API, as part of IPNS, the name system IPFS uses. It is also used in a variety of tools around IPFS.

Learn more at the [IPFS and DNSLink documentation](https://docs.ipfs.io/concepts/dnslink/).

#### IPFS Gateway

The IPFS Gateway resolves DNSLinks automatically. See [gateway recipes](https://github.com/ipfs/go-ipfs/blob/master/docs/config.md#gateway-recipes).

#### IPFS Companion

[IPFS Companion](https://github.com/ipfs-shipyard/ipfs-companion) can resolve DNSLinks automatically. See more in the [IPFS Companion documentaion](https://docs.ipfs.io/how-to/dnslink-companion/).

#### add yours here

Add yours on github.

### Websites

Many websites use DNSLink and IPFS. Check some of them out:

- https://ipfs.io
- https://filecoin.io
- https://protocol.ai
- https://libp2p.io
- https://multiformats.io
- _add yours here_

## Best Practices

### Set a low `TTL` in the `TXT` record.

The default `TTL` for DNS records is usually `3600`, which is 1hr. We recommend setting a low `TTL` in the `TXT` record, something like `60` seconds or so. This makes it so you can update your name quickly, and use it for website deploys.

## FAQ

#### Is there an IETF RFC spec?

Not yet. We should write one.

#### Can I use DNSLink in non-DNS systems?

Yes absolutely. For example, you can use DNSLink to resolve names from Ethereum Name System (ENS) thanks to DNS-interop provided at https://eth.link. If you use DNSLink for something cool like this, be sure to add it to the [Users](#Users) section of this doc.

#### Why use `_dnslink.domain` instead of `domain`?

`_dnslink.domain` does not conflict with `CNAME` and `ALIAS` records, and is better for operations security: enables you to set up an automated publishing or delegate control over your DNSLink records to a third party without giving away full control over the original DNS zone.

`TXT` record on `domain` is deprecated and resolved by some software only for legacy reasons: we historically started with supporting records on the normal domain, and only found the issue with `CNAME` and `ALIAS` records later on. Those problems are absent when `_dnslink.domain` is used.

Rule of thumb: always use `_dnslink.domain`.

#### Why use the `dnslink=` prefix in the `TXT` value?

The prefix `dnslink=` is there to signal that this `TXT` record value is a DNSLink. This is important because many systems use TXT records, and there is a convention of storing multiple space separated values in a single `TXT` record. Following this format allows your DNSLink resolver to parse through whatever is in the `TXT` record and use the first entry prefixed with `dnslink=`.

#### Why not use other DNS record types, like `SRV`?

Special purpose records enforce some structure to their values, and these are not flexible enough. We wanted a simple protocol that allowed us to link to any other system or service, in perpetuity. This can only be achieved with systems designed to allow structure within the input. The `TXT` record is a good example of this -- it imposes a string value, but it allows the user to make sense out of anything within that string. Sets of users can imposese different rules for resolution. In our case, this is handled through [Multiaddr](https://multiformats.io/multiaddr).

#### Why not just extend DNS with more record types?

It seems that what DNSLink does could be accomplished by a verity of new special purpose record types. But extending DNS is a very expensive process -- in terms of time and adoption. We wanted to find a way to use all the existing infrastructure without requiring any changes at all.


#### Why DNS?

> The [Domain Name System](https://en.wikipedia.org/wiki/Domain_Name_System) (DNS) is a hierarchical and decentralized naming system for computers, services, or other resources connected to the Internet or a private network. ... By providing a worldwide, distributed directory service, the Domain Name System has been an essential component of the functionality of the Internet since 1985.

DNS is:
- **somewhat decentralized** - it delegates naming authority to nations and organizations world-wide
- **hierarchical** - it achieves scalable naming via name space partitioning
- **scalable** - it is the dominant name system on the planet -- most internet traffic uses it
- **ubiquitous** - as the dominant internet name system since 1985, DNS is well-known and well-supported.

As a name system, DNS has pragmatic solutions for:
- **control of names**
- **scalable naming**
- **scalable reads and writes**
- **partition tolerance**
- **and more**

Of course, DNS has shortcomings:
- **somewhat centralized** - central authorities manage DNS roots (ICANN, etc.)
- **censorable** - DNS is easy to censor, at the registrar and ISP levels
- **temporary** - DNS names are _rented_, not really _owned_, and can be revoked and seized
- **not offline-first** - DNS does not have good solutions for values to spread through intermittently connected networks
- **cumbersome** - buying, registering, and setting values on a name is more complicated than acquiring a mutable pointer should be.
- **not program friendly** - registering DNS names requires being a human or a human organization with human agents, with access to credit cards. this isn't suitable for lots of programs.

These are all good reasons to develop new, better name systems. But until those work better than DNS, you'll probably use DNS.

## Contribute

The DNSLink project is organized through github's ["dnslink-std" organization](https://github.com/dnslink-std/community).


### Contributors

- [@jbenet](https://github.com/jbenet)
- [@whyrusleeping](https://github.com/whyrusleeping)
- [@lgierth](https://github.com/lgierth)
- [@victorbjelkholm](https://github.com/victorbjelkholm)
- [@diasdavid](https://github.com/diasdavid)
- [@martinheidegger](https://github.com/martinheidegger)
- and undoubtedly many more

Developed with the support of [Protocol Labs](https://protocol.ai)

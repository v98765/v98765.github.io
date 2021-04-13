## looking glass

[rfc7999](https://tools.ietf.org/html/rfc7999) blackhole community

AS | LG | BlackHole community
---|---|---
1299 | [lg.telia.net](https://lg.telia.net/) | 1299:999
8631 | [msk-ix.ru/lookingglass/](https://msk-ix.ru/lookingglass/) | 65535:666
9002 |[lg.retn.net](https://lg.retn.net/) | 9002:666
49869 | [lg.piter-ix.ru](https://lg.piter-ix.ru/) | 50509:666
56931 | [www.eurasiapeering.com/looking-glass/](https://www.eurasiapeering.com/looking-glass/) | 65535:666
57724 | | 
199599 | [lg.cirex.ru](http://lg.cirex.ru/) | 64000:666



[mastertel](http://lg.mastertel.ru/), [mts](http://lg.mtu.ru/) 

У РЕТН есть усуга extended blackhole, где с 9002:667 фильтруется весь udp, с 9002:668 фильтрация UDP-трафика от «известных amplifier’ов» (UDP with source-port 19, 53, 123, 161, 389, 520, 1900, 11211) и фрагментированного UDP-трафика


## route info

[bgpview.io](https://bgpview.io) и его [api](https://bgpview.docs.apiary.io/), например `curl https://api.bgpview.io/asn/as_number`.

[ifconfig.co](http://ifconfig.co/). В какой автономной системе хост `curl ifconfig.co/asn` и его публичный ip-адрес `curl ifconfig.co`

## ripe

[rpki-validator.ripe.net/roas](https://rpki-validator.ripe.net/roas)
[stat.ripe.net](https://stat.ripe.net/) и [beta](https://beta-ui.stat.ripe.net/launchpad)

We are pleased to announce that we have updated the RIPE NCC Certification Practice Statement (CPS) for the Resource Public Key Infrastructure (RPKI).
As a Certificate Authority in RPKI, we are required to publish a CPS. In this document, we describe:

* How certificates are issued, managed, and revoked;
* How we perform the key management;
* The kind of certificates we issue.

The CPS is an update to RIPE-549 and is published as RIPE-751:
[www.ripe.net/publications/docs/ripe-751](https://www.ripe.net/publications/docs/ripe-751/)

The document follows the "Template for a CPS for the RPKI" as defined in RFC7382.

## Прочее

[peeringdb.com](http://peeringdb.com/)

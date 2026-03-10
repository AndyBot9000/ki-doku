# Eigenes Autonomous System betreiben: BGP auf FreeBSD mit FRR, GRE-Tunneln und Policy Routing

!!! info "Quelle"
    Originalartikel: [Running Your Own AS: BGP on FreeBSD with FRR, GRE Tunnels, and Policy Routing](https://blog.hofstede.it/running-your-own-as-bgp-on-freebsd-with-frr-gre-tunnels-and-policy-routing/) von Christian Hofstede-Kuhn (Larvitz Blog)
    Veröffentlicht: 8. Februar 2026 · Übersetzt von Andy, 10. März 2026

---

Ein eigenes Autonomous System (AS) im öffentlichen Internet zu betreiben klingt nach etwas, das ISPs und großen Unternehmen vorbehalten ist. Ist es nicht. Mit sponsoring LIRs, die AS-Nummern und IPv6-Präfixe auch für Einzelpersonen zugänglich machen, und FreeBSD als Routing-Plattform, kann man eigenen Adressraum von einer einzigen virtuellen Maschine aus in der Default-Free Zone (DFZ) ankündigen.

Dieser Artikel beschreibt das vollständige Setup: Ressourcen von RIPE via sponsoring LIR beschaffen, einen FreeBSD-BGP-Router mit FRR konfigurieren, GRE/GIF-Tunnel bauen, um Präfixe auf entfernte Server zu verteilen – und das Routing-Problem lösen, das entsteht, wenn ein Server gleichzeitig aus zwei verschiedenen IPv6-Adressräumen kommunizieren muss.

!!! note "Hinweis zu Adressen"
    Alle provider-zugewiesenen IP-Adressen, Hostnamen und Management-IPs wurden durch RFC 5737 / RFC 3849 Dokumentationsranges ersetzt. Die eigene AS-Nummer (AS201379) und das Präfix (2a06:9801:1c::/48) sind öffentliche BGP-Ressourcen und werden im Original angezeigt.

---

## Warum ein eigenes AS betreiben?

Provider-zugewiesene IPv6-Adressen sind an diesen Provider gebunden. Wechselt man den Hoster, ändern sich die Adressen – zusammen mit DNS-Einträgen, Firewall-Regeln, Reputation und allem, was darauf referenziert. Mit eigenem AS und Präfix **folgen die Adressen einem selbst**. Server migrieren, Tunnel-Endpunkt aktualisieren – und der Traffic fließt wieder, ohne eine einzige Service-Konfiguration anzufassen.

Es gibt auch weniger pragmatische Gründe: BGP zu verstehen verändert grundlegend, wie man über Internet-Routing nachdenkt. Das eigene Präfix auf Looking-Glasses weltweit zu sehen ist schlicht befriedigend.

---

## Ressourcen beschaffen

Um Präfixe im Internet ankündigen zu können, braucht man zwei Dinge vom Regional Internet Registry (in Europa: RIPE NCC):

- Eine **AS-Nummer** – die eigene Identität in BGP
- Ein **IPv6-Präfix** – der Adressraum, der angekündigt wird

Als Privatperson muss man kein RIPE-Mitglied werden (was Gebühren und Bürokratie bedeutet). Stattdessen arbeitet man mit einem **sponsoring LIR** – einem bestehenden RIPE-Mitglied, das die Ressourcen-Registrierung sponsert. Der Prozess umfasst typischerweise:

1. Antragsformular mit dem geplanten Verwendungszweck ausfüllen
2. RIPE-Datenbank-Objekte erstellen (`aut-num`, `inet6num`, `route6`)
3. **RPKI ROAs** (Route Origin Authorizations) einrichten – kryptografische Bindung des Präfixes an die AS-Nummer

---

## Architektur-Überblick

Das Setup hat zwei Ebenen: ein BGP-Router, der mit Upstream-Providern peert, und nachgelagerte Server, die getunnelte Subnetze aus dem /48 erhalten.

```
                    ┌──────────────────────────────┐
                    │     Default-Free Zone         │
                    └──────┬──────────────┬─────────┘
                           │              │
                    AS34927 (iFog)   AS209735 (Lagrange)
                           │              │
                      GRE-Tunnel     Direktes Peering
                           │              │
                    ┌──────┴──────────────┴─────────┐
                    │    router01 (BGP-Router)       │
                    │     FreeBSD + FRR              │
                    │     AS201379                   │
                    │     2a06:9801:1c::/48          │
                    └──────┬──────────────┬─────────┘
                           │              │
                      GIF-Tunnel     GIF-Tunnel
                      (Proto 41)     (Proto 41)
                           │              │
                    ┌──────┴───┐   ┌──────┴──────────┐
                    │  vps01   │   │  dcgw01          │
                    │  VPS     │   │  DC OPNsense     │
                    │  :1000:  │   │  :2000::/62      │
                    │  /64     │   │                  │
                    └──────────┘   └──────────────────┘
```

Der BGP-Router (`router01`) kündigt `2a06:9801:1c::/48` bei zwei Upstream-Providern an. Einzelne `/64`s (und ein `/62` für das Colocation-Datacenter) werden via GIF-Tunnel (IPv6-in-IPv4) zu nachgelagerten Servern getunnelt.

---

## Der BGP-Router

### Netzwerk-Konfiguration (`/etc/rc.conf`)

```sh
hostname="router01"

kern_securelevel_enable="YES"
kern_securelevel="2"

# Physisches Interface
ifconfig_vtnet0="inet 198.51.100.10/24 -rxcsum -txcsum -rxcsum6 -txcsum6 -lro -tso"
ifconfig_vtnet0_ipv6="inet6 2001:db8:100::96/64"

defaultrouter="198.51.100.1"
ipv6_defaultrouter="2001:db8:100::1"

# Loopback-Alias für das eigene Präfix
ifconfig_lo0_alias0="inet6 2a06:9801:1c::1 prefixlen 64"

# Tunnel-Interfaces
cloned_interfaces="gif0 gif1 gre0"
kld_list="if_gif if_gre"

# GRE-Tunnel zum Transit-Provider (iFog)
ifconfig_gre0="tunnel 198.51.100.10 198.51.100.44"
ifconfig_gre0_ipv6="inet6 2001:db8:300::2 2001:db8:300::1 prefixlen 128"

# GIF-Tunnel zum VPS
ifconfig_gif0="tunnel 198.51.100.10 203.0.113.10"
ifconfig_gif0_ipv6="inet6 2a06:9801:1c:ffff::1 2a06:9801:1c:ffff::2 prefixlen 128"
ipv6_route_cloud="2a06:9801:1c:1000::/64 2a06:9801:1c:ffff::2"

# GIF-Tunnel zum Datacenter
ifconfig_gif1="tunnel 198.51.100.10 192.0.2.50"
ifconfig_gif1_ipv6="inet6 2a06:9801:1c:ffff::3 2a06:9801:1c:ffff::4 prefixlen 128"
ipv6_route_dc="2a06:9801:1c:2000::/62 2a06:9801:1c:ffff::4"

# Blackhole-Route für das Aggregat + Downstream-Routen
ipv6_static_routes="myblock cloud dc"
ipv6_route_myblock="2a06:9801:1c::/48 -reject"
ipv6_gateway_enable="YES"

pf_enable="YES"
frr_enable="YES"
```

**Wichtige Details:**

- **Blackhole-Route** (`-reject` für das /48): Ohne sie würde Traffic für nicht zugewiesene Subnetze der Default-Route folgen – zurück zum Upstream, was eine Routing-Schleife erzeugt. Die Blackhole verwirft solchen Traffic lokal.
- **Point-to-Point-Tunnel-Adressen** mit `/128`-Präfixen aus dem `2a06:9801:1c:ffff::/64`-Link-Subnetz.
- **GRE vs. GIF**: Der iFog-Peering-Tunnel nutzt GRE (Anforderung des Providers). Downstream-Tunnel nutzen GIF (Proto 41, IPv6-in-IPv4) – einfacher, weniger Overhead.

### FRR-Konfiguration (`/usr/local/etc/frr/frr.conf`)

```
frr version 10.5.1
frr defaults traditional
hostname router01

ipv6 prefix-list PL-MY-NET seq 5 permit 2a06:9801:1c::/48

ipv6 prefix-list PL-BOGONS seq 5 deny ::/0 le 7
ipv6 prefix-list PL-BOGONS seq 10 deny ::/8
# ... (vollständige Bogon-Filter-Liste) ...
ipv6 prefix-list PL-BOGONS seq 95 deny ff00::/8
ipv6 prefix-list PL-BOGONS seq 100 deny 2a06:9801:1c::/48
ipv6 prefix-list PL-BOGONS seq 105 deny ::/0 ge 49
ipv6 prefix-list PL-BOGONS seq 110 permit ::/0 le 48

route-map RM-IFOG-OUT permit 10
 match ipv6 address prefix-list PL-MY-NET
 set community 34927:9501 34927:9301 additive

route-map RM-LAGRANGE-OUT permit 10
 match ipv6 address prefix-list PL-MY-NET
 set as-path prepend 201379 201379

route-map RM-IFOG-IN permit 10
 match ipv6 address prefix-list PL-BOGONS

route-map RM-LAGRANGE-IN permit 10
 match ipv6 address prefix-list PL-BOGONS

router bgp 201379
 bgp router-id 198.51.100.10
 no bgp default ipv4-unicast
 neighbor 2001:db8:300::1 remote-as 34927
 neighbor 2001:db8:300::1 ttl-security hops 1
 neighbor 2001:db8:100::ff remote-as 209735
 neighbor 2001:db8:100::ff ttl-security hops 1
 !
 address-family ipv6 unicast
  network 2a06:9801:1c::/48
  neighbor 2001:db8:300::1 activate
  neighbor 2001:db8:300::1 route-map RM-IFOG-IN in
  neighbor 2001:db8:300::1 route-map RM-IFOG-OUT out
  neighbor 2001:db8:100::ff activate
  neighbor 2001:db8:100::ff route-map RM-LAGRANGE-IN in
  neighbor 2001:db8:100::ff route-map RM-LAGRANGE-OUT out
 exit-address-family
```

**Wichtige Design-Entscheidungen:**

| Konfiguration | Zweck |
|---|---|
| `PL-MY-NET` | Nur das eigene /48 – verhindert, dass andere Präfixe angekündigt werden |
| `PL-BOGONS` | Aggressiver Eingangsfilter – verwirft nicht-routbaren Adressraum, eigenes Präfix (Loop-Schutz), zu spezifische (>/48) und zu grobe (<8) Präfixe |
| `RM-IFOG-OUT` mit Communities | Steuert via iFog-eigene Communities, welchen Peers das Präfix weitergegeben wird |
| `RM-LAGRANGE-OUT` mit AS-Path-Prepending | Macht den Lagrange-Pfad länger → Internet bevorzugt iFog-Pfad (Traffic Engineering) |
| `no bgp default ipv4-unicast` | Rein IPv6 – kein IPv4-Address-Family-Aktivierung |
| `ttl-security hops 1` | GTSM: Verwirft BGP-Pakete mit TTL < 254 – schützt vor Remote-Angriffen auf die BGP-Session |
| `maximum-prefix 250000 90 restart 30` | Sicherheitsventil gegen Route Leaks vom Upstream |

---

## Der nachgelagerte Server: Dual-Stack mit Policy Routing

Hier wird es interessant. Der VPS (`vps01`) hat bereits provider-zugewiesenes IPv6 vom Hoster. Jails auf diesem Server nutzen Adressen aus **zwei Adressräumen gleichzeitig**:

- **Provider-IPv6** (`2001:db8:200::.../68`) – die Hoster-Adressen, per NAT am Host
- **BGP-IPv6** (`2a06:9801:1c:1000::/64`) – das eigene Präfix, nativ via GIF-Tunnel geroutet

**Das Problem:** Sendet ein Jail Traffic von seiner BGP-Adresse, muss dieser Traffic durch den GIF-Tunnel zum BGP-Router – nicht über die Default-Route zum VPS-Provider (wo er als spoofed verworfen würde). Provider-Traffic muss aber weiterhin die normale Default-Route nutzen.

**Die Lösung: Dual-FIB Policy Routing** – FreeBSDs Implementation mehrerer Routing-Tabellen.

### Wie Dual-FIB funktioniert

FreeBSD unterstützt mehrere Routing-Tabellen, sogenannte **FIBs** (Forwarding Information Bases). Jede FIB ist eine unabhängige Routing-Tabelle mit eigener Default-Route.

```
FIB 0 (Standard):
  default → vtnet0 → VPS-Provider-Upstream
  Genutzt von: Host-Traffic, Provider-adressierter Jail-Traffic

FIB 1:
  default → gif0 → BGP-Router (router01)
  Genutzt von: BGP-adressierter Jail-Traffic (2a06:9801:1c::/48)
```

### Netzwerk-Konfiguration des Servers

```sh
# GIF-Tunnel zum BGP-Router – zugewiesen zu FIB 1
ifconfig_gif0="fib 1 tunnel 203.0.113.10 198.51.100.10 tunnelfib 0"
ifconfig_gif0_ipv6="inet6 2a06:9801:1c:ffff::2 2a06:9801:1c:ffff::1 prefixlen 128"

# FIB 1 Routing-Tabellen-Einträge
static_routes="fib1default jailleak bgplink"
route_fib1default="-6 default -interface gif0 -fib 1"
route_jailleak="-6 2001:db8:200:0:1000::/68 -interface bastille0 -fib 1"
route_bgplink="-6 2a06:9801:1c:1000::/64 -interface bastille0 -fib 1"
```

Die GIF-Tunnel-Zeile enthält zwei kritische Direktiven:

- **`fib 1`**: Das Tunnel-Interface selbst lebt in FIB 1
- **`tunnelfib 0`**: Aber die äußere IPv4-Kapselung (der `203.0.113.10 → 198.51.100.10`-Wrapper) nutzt FIB 0 – die IPv4-Route zum BGP-Router läuft über die Provider-Default-Route. Ohne `tunnelfib 0` würden die gekapselten Pakete FIB 1's Default-Route nutzen (die auf gif0 zeigt) → rekursive Schleife.

### PF: Der Routing-Klebstoff

PF trifft die adressbasierte Routing-Entscheidung. Wenn ein Jail ein Paket von einer BGP-Adresse sendet, weist PF es FIB 1 zu:

```pf
# BGP-adressierter Jail-Traffic → in Routing-Tabelle 1 erzwingen
pass in quick on bastille0 inet6 from $bgp_net to any rtable 1 keep state
```

`rtable 1` weist PF an, matchende Pakete über FIB 1 zu routen. Da FIB 1's Default-Route auf gif0 zum BGP-Router zeigt, werden diese Pakete korrekt getunnelt.

Für eingehenden Traffic via Tunnel sorgt `reply-to` dafür, dass Return-Traffic denselben Pfad nimmt:

```pf
# Eingehender BGP-Traffic – reply-to stellt korrekte Rückroute sicher
pass in quick on $tun_if reply-to ($tun_if $bgp_hub_ip) inet6 \
    from any to $bgp_net keep state
```

Ohne `reply-to` würde der Kernel FIB 0 für Return-Traffic nutzen – Antworten würden über vtnet0 mit falscher Source-Routing gesendet und vom Provider als spoofed verworfen.

---

## Vollständiger Traffic-Fluss

**Eingehende Verbindung zu einem Jail:**

```
1. Client sendet Paket an 2a06:9801:1c:1000::10
2. Paket traversiert Internet → AS201379 via iFog oder Lagrange
3. router01 leitet weiter durch gif0-Tunnel → vps01
4. vps01 empfängt Proto 41 auf vtnet0, dekapsuliert → gif0
5. PF matched: reply-to (gif0, bgp_hub_ip), erstellt State
6. Paket → bastille0 → Jail
7. Jail antwortet, Paket verlässt bastille0
8. PF's State-Table: reply-to → sende via gif0 zu bgp_hub_ip
9. gif0 kapselt (Proto 41) über FIB 0 → router01
10. router01 empfängt, leitet weiter → Upstream → Internet → Client
```

**Ausgehende Verbindung des Jails (BGP-Adresse):**

```
1. Jail sendet von 2a06:9801:1c:1000::10
2. Paket kommt auf bastille0 an
3. PF matched: "from $bgp_net → rtable 1"
4. Kernel routet via FIB 1 → Default-Route → gif0
5. gif0 kapselt über FIB 0 → vtnet0 → router01
6. router01 leitet weiter → Internet (Source: 2a06:9801:1c:1000::10)
```

Gleichzeitig kann dasselbe Jail über seine Provider-Adresse und die normale FIB 0 kommunizieren. **Beide Adressräume koexistieren transparent** – unterschieden allein durch PF-Regeln und FIB-Auswahl.

---

## Verifikation

Aus einem Jail mit beiden Adressen:

```bash
# Traffic von Provider-Adresse – per NAT durch den Hoster
root@caddy:~ # curl --interface 2001:db8:200:0:1000::10 https://ifconfig.co
2001:db8:200::2   ← NAT-Adresse des Hosts

# Traffic von BGP-Adresse – nativ durch den Tunnel geroutet
root@caddy:~ # curl --interface 2a06:9801:1c:1000::10 https://ifconfig.co
2a06:9801:1c:1000::10   ← Echte BGP-Adresse!
```

---

## Lessons Learned

| Lesson | Details |
|---|---|
| **MSS-Clamping ist Pflicht** | Jede Tunnelschicht reduziert die MTU. GIF: −20 Bytes (IPv4-Header), GRE: noch mehr. Ohne MSS-Clamping in PF's `scrub`-Regeln: mysteriöse Verbindungsabbrüche bei großen Paketen |
| **FIB-Trennung ist sauberer als Hacks** | FreeBSDs Multi-FIB ist ein First-Class-Feature. `rtable` in PF + `fib`/`tunnelfib` auf Interfaces geben volle Kontrolle – konzeptuell klarer als Linux-Alternativen mit `ip6tables MARK` |
| **Bogon-Filterung ist wichtig** | Auch für kleine Netze. Das Internet ist voller Fehlkonfigurationen. Inbound-Routes aggressiv filtern verhindert, dass der Router Unsinn akzeptiert |
| **`reply-to` löst asymmetrisches Routing** | Wenn Traffic auf mehreren Interfaces ankommen kann, wählt der Kernel möglicherweise den falschen Return-Pfad. `reply-to` erzwingt die Antwort über das Ankunfts-Interface |
| **Zwei Upstreams von Anfang an** | Ein einziger Upstream = keine Redundanz, kein Traffic Engineering. Zwei Upstreams ermöglichen Failover und AS-Path-Prepending |

---

## Fazit

Ein eigenes AS im Internet zu betreiben ist zugänglicher als die meisten annehmen. Die Hürde ist nicht technische Komplexität – es ist das Wissen, dass die Option existiert. Eine FreeBSD-VM, FRR, ein paar Tunnel und sorgfältige PF-Regeln geben provider-unabhängige Adressierung, echtes BGP-Peering und ein tieferes Verständnis, wie das Internet wirklich funktioniert.

Der **Dual-FIB-Ansatz** auf dem nachgelagerten Server ist die eleganteste Lösung: BGP-Traffic nimmt den Tunnel, Provider-Traffic die Default-Route, PF's `rtable`-Direktive trifft die Entscheidung rein auf Basis der Source-Adresse. Beide Pfade koexistieren transparent – die Jails müssen nichts vom Routing darunter wissen.

> *„Das Internet ist ein Netzwerk von Netzwerken – und jetzt bist du eines davon."* – Christian Hofstede-Kuhn

---

## Referenzen

- [RIPE NCC – Ressourcen beantragen](https://www.ripe.net/manage-ips-and-asns/resource-management/contract-and-registration-services/independent-resources/)
- [FRR Dokumentation](https://docs.frrouting.org/)
- [FreeBSD Handbook: Firewalls (PF)](https://docs.freebsd.org/en/books/handbook/firewalls/)
- [FreeBSD setfib(1)](https://man.freebsd.org/cgi/man.cgi?setfib)
- [bgp.tools – BGP Looking Glass](https://bgp.tools/)
- [RIPE RPKI Dokumentation](https://www.ripe.net/manage-ips-and-asns/resource-management/rpki/)
- [RFC 5082 – GTSM (TTL Security)](https://www.rfc-editor.org/rfc/rfc5082)

---

*Originalartikel: [blog.hofstede.it](https://blog.hofstede.it/running-your-own-as-bgp-on-freebsd-with-frr-gre-tunnels-and-policy-routing/) · Autor: Christian Hofstede-Kuhn · Übersetzt von Andy, 10.03.2026*

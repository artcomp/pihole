# D-Link DIR-809 — Informações do Roteador

> Coletado via interface web (`http://192.168.0.1`) em 19/10/2026.
> Login: `admin` / senha em branco.

---

## Identificação do dispositivo

| Campo | Valor |
|---|---|
| Fabricante | D-Link International |
| Modelo | DIR-809 |
| Título da UI | D-LINK SYSTEMS, INC. \| WIRELESS ROUTER |
| Hardware | A2 |
| Firmware | 1.07 |
| Servidor web | eCos Embedded Web Server (porta 80) |
| MAC (WAN) | `90:8d:78:63:02:ff` (OUI D-Link) |

---

## Topologia / modo de operação

O DIR-809 está operando em **modo roteador (NAT duplo)** atrás do roteador do ISP.

```
ISP (192.168.2.1)  ← WAN do DIR-809 (192.168.2.7)
                         ↓
                   DIR-809 (LAN 192.168.0.1)
```

- **Rede LAN do DIR-809:** `192.168.0.0/24`
- **Gateway LAN:** `192.168.0.1`
- **SSID 2.4 GHz:** `Quarto-wifi`
- **SSID 5 GHz:** `Quarto-wifi-5GHz`

> O motivo de não ser acessível pela rede principal (`192.168.2.x`): o IP `192.168.2.7` é a interface WAN do DIR-809 e todas as portas estão filtradas pelo firewall. O painel admin só responde do lado LAN (`192.168.0.x`).

---

## WAN (lado de conexão com o ISP)

Coletado de `wan_connection_status.asp`:

| Campo | Valor |
|---|---|
| Tipo de WAN | DHCP (`wanType=autoIp`, `dsl_mode=1`) |
| IP WAN | `192.168.2.7` |
| Máscara | `255.255.255.0` |
| Gateway | `192.168.2.1` |
| DNS primário | `192.168.0.101` (Pi-hole — campo manual; antes: `177.152.174.16` do ISP) |
| DNS secundário | `0.0.0.0` (campo manual — sem fallback; antes: `177.152.173.35` do ISP) |
| Estado da conexão | `connection_state=2` (conectado) |
| Link físico | `wan_physical_link=0` / `wan_physical_state=0` (sem cabo na porta WAN do DIR-809 — conectado via Wi-Fi/repetidor) |
| Uptime da interface | ~2 s (reiniciado recentemente) |

---

## LAN (lado local)

| Campo | Valor |
|---|---|
| IP do roteador | `192.168.0.1` |
| Máscara | `255.255.255.0` |
| Gateway LAN | `0.0.0.0` (roteador é o gateway) |
| Domínio local | `dlinkrouter` |
| Relay DNS | **Desabilitado** (configurado em 19/10/2026) |
| Servidor DHCP | Habilitado |
| Faixa DHCP | `192.168.0.100` – `192.168.0.199` |
| Lease DHCP | 1440 min (24 h) |
| Always broadcast | Habilitado |

---

## DNS automático via Pi-hole (configurado em 19/10/2026)

O DIR-809 entrega o Pi-hole (`192.168.0.101`) como DNS primário para **todos** os clientes DHCP da rede Quarto-wifi, sem configurar nada nos dispositivos.

Como foi feito (o firmware ignora DNS fora do subnet LAN no modo DHCP-WAN e não tem campo de DNS próprio no DHCP):

1. **WAN primário DNS** = `192.168.0.101`, **secundário `0.0.0.0`** (`Basic/wan_dhcp.asp`, campos "Servidor DNS primário/secundário (opcional)")
2. **DNS relay desabilitado** (`Basic/Network.asp`) — sem relay, o DHCP anuncia o DNS WAN aos clientes
3. **Reserva DHCP** do Pi: `pi` / `192.168.0.101` / `2c:cf:67:1c:c0:f3` / expiração "Nunca"

Resultado do lease DHCP: `domain_name_server = {192.168.0.101}` — o
`0.0.0.0` é descartado como inválido pelo D-Link no anúncio DHCP
(confirmado em `ipconfig getpacket en0`).

> **Por que o DNS secundário é `0.0.0.0` (e não um resolver público):**
> quando `1.1.1.1` (ou `8.8.8.8`) aparece na lista de resolvers do
> macOS, o Chrome em modo "Usar DNS seguro" (`chrome://settings/security`)
> faz upgrade automático para DoH usando o resolver do sistema — neste
> caso, Cloudflare/Google DoH — e **contorna o Pi-hole** (anúncios
> carregam via DoH enquanto `curl` ainda é bloqueado). Com `0.0.0.0` no
> campo (ou vazio), não há resolver público no lease e todas as
> consultas passam pelo Pi-hole.
>
> **Histórico:** em 19/10/2026 o Pi-hole caiu e `8.8.8.8` foi
> adicionado como secundário p/ restaurar a internet. Com o Pi-hole
> de volta, foi substituído por `0.0.0.0`.
>
> **Trade-off:** se o Pi-hole cair, não há DNS fallback — a resolução de nomes para. (Qualquer resolver público secundário reativaria o bypass de DoH do Chrome; desativar "Usar DNS seguro" no Chrome é alternativa possível caso precise do fallback.)

> Limitação conhecida do firmware: `config.wan_force_static_dns_servers` não persiste como `true` via POST direto; a UI (Playwright) gravou o valor normalmente e ele persistiu após reboot.

---

## Blocklists do Pi-hole (configuradas em 19/10/2026)

**11 listas confiáveis**, todas com URLs dinâmicas (atualização automática sem intervenção manual):

| # | Lista | Domínios | Destaque |
|---|---|---|---|
| 1 | StevenBlack hosts | 71k | Base unificada |
| 3 | OISD full | 235k | Curada, inclui Google/Meta ads |
| 4 | AdGuard DNS filter | 188k | Forte cobertura Google/Meta |
| 5 | AdAway | 6.5k | Clássica hosts |
| 6 | 1hosts Lite | 203k | DNS-safe, ads |
| 7 | notracking | 500k | Rastreamento abrangente |
| 8 | Dandelion Sprout Anti-Malware | 12k | Malware/ads |
| 10 | MVPS hosts | 8.7k | Clássica |
| 11 | Dan Pollock someonewhocares | 13k | Clássica |
| 12 | pgl.yoyo adservers | 3.6k | Adservers |
| 13 | HaGeZi Pro (adblock) | 203k | Google Ads, DoubleClick, Meta/Facebook, trackers |
| 14 | Phishing Army hosts | 156k | Phishing |

**Total: ~1.12M domínios únicos** bloqueados (gravity: 1.6M com subdomínios).

**Atualização automática:** cron do Pi-hole roda `pihole updateGravity` todo domingo 03:30 — as listas são URLs dinâmicas (GitHub raw, OISD, etc.), então sempre baixam a versão mais recente.

**Domínios de anúncio Google/Meta verificados como bloqueados:**
- `doubleclick.net`, `pagead2.googlesyndication.com`, `googlesyndication.com`, `googleadservices.com`, `ad.doubleclick.net`, `securepubads.g.doubleclick.net`, `google-analytics.com`, `googletagmanager.com` → todos `0.0.0.0`
- `an.facebook.com` (Audience Network), `pixel.facebook.com`, `facebook.net` → `0.0.0.0` / NXDOMAIN
- `facebook.com` principal **não** bloqueado (site continua acessível; só os subdomínios de ads/tracking são bloqueados)
- `pubmatic.com`, `rubiconproject.com`, `criteo.com`, `taboola.com`, `outbrain.com`, `amazon-adsystem.com` → todos `0.0.0.0`
- CNAME deep-inspect **habilitado** — bloqueia até via CNAME chain

**Cobertura de anúncios first-party:** alguns anúncios de sites (ex: Globo usa `ads-library.globo.com` + `rec.ads.globo.com` em domínio próprio) não são bloqueados por listas genéricas. No teste com ge.globo.com via Chromium: requisições a `securepubads.g.doubleclick.net` e `ads-library.globo.com` falharam (`net::ERR_CONNECTION_REFUSED`), espaço de anúncio ficou vazio.

---

## Wireless

### 2.4 GHz — `Quarto-wifi`

| Campo | Valor |
|---|---|
| SSID | `Quarto-wifi` |
| Canal | 6 |
| Segurança | WPA2-Personal (security_type=6) |
| Ocultar SSID | Não |

### 5 GHz — `Quarto-wifi-5GHz`

| Campo | Valor |
|---|---|
| SSID | `Quarto-wifi-5GHz` |
| Canal | 153 |
| Segurança | WPA2-Personal (security_type=6) |
| Ocultar SSID | Não |

---

## Clientes DHCP conectados

| IP | MAC | Hostname | Reserva | Restante do lease |
|---|---|---|---|---|
| 192.168.0.102 | `ee:dc:65:d9:b6:b0` | Unknow | Não | ~84897 s (~23.6 h) |
| 192.168.0.101 | `de:77:48:66:a9:bd` | Unknow | Não | ~82504 s (~22.9 h) |

> `192.168.0.102` é o Mac deste dispositivo (conectado à "Quarto-wifi").

---

## Funcionalidades disponíveis na UI

- **Setup (Básico):** Internet (WAN), Wireless, LAN, Tempo, Filtro de sites, IPv6
- **Avançado:** Servidor virtual (port forwarding), Aplicações especiais, Filtro MAC, Filtro URL, QoS, Firewall, Wireless avançado, Rede avançada, Roteamento, WPS
- **Ferramentas:** Admin, Sistema, Tempo, Firmware, DDNS, System Check, Schedules, SysLog
- **Status:** Device Info, Wireless, WAN connection status, DHCP clients

---

## Credenciais

| Campo | Valor |
|---|---|
| Usuário | `admin` |
| Senha | *(em branco — padrão de fábrica)* |
| IP do painel | `http://192.168.0.1` |

> **Recomendação:** troque a senha admin e o SSID/WPA2 antes de qualquer uso permanente. A senha em branco é o padrão de fábrica e representa um risco de segurança.

---

## Notas técnicas

- **Autenticação do painel:** HMAC-MD5 com challenge/cookie via `formRequestKey.htm` + `formLogin.htm`; requer os campos `curTime` e `curUid` no POST.
- **Fonte dos dados:** endpoints `wan_connection_status.asp`, `dhcp_clients.asp`, páginas `Basic/*.asp`, `Status/Device_Info.asp`.
- **Rede principal (ISP):** `192.168.2.0/24` — gateway `192.168.2.1` (Stephen Technologies ONT/router).
- O DIR-809 não é acessível da rede `192.168.2.x` porque o firewall bloqueia todas as portas na interface WAN. É necessário estar na rede `192.168.0.x` (conectado a "Quarto-wifi" ou por cabo na LAN).

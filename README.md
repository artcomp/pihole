# Pi-hole — Bloqueio de Anúncios Automático para Toda a Rede Quarto-wifi

> Bloqueio de anúncios e rastreadores para **todos os dispositivos** da rede
> `192.168.0.0/24` ("Quarto-wifi") automaticamente via DHCP — **nenhuma
> configuração por dispositivo**. Documentado para reprodução total.

---

## Visão Geral da Arquitetura

```
                    Internet
                       │
            ┌──────────┴──────────┐
            │  ISP Router         │
            │  192.168.2.1        │
            │  (rede "Domingues") │  ← NÃO TOCADO
            └──────────┬──────────┘
                       │ WAN (192.168.2.7)
            ┌──────────┴──────────┐
            │  D-Link DIR-809     │
            │  LAN: 192.168.0.1   │
            │  SSID: Quarto-wifi  │
            │  DHCP: .100–.199    │
            │  DNS Relay: OFF     │
            │  DNS WAN → Pi-hole  │
            └──────────┬──────────┘
                       │ LAN (192.168.0.0/24)
         ┌─────────────┼─────────────┐
         │             │             │
   ┌─────┴─────┐ ┌─────┴─────┐ ┌─────┴─────┐
   │  Pi-hole  │ │  Mac      │ │  Celular  │
   │  .101     │ │  .102     │ │  (DHCP)   │
   │  DNS:53   │ │  DNS→.101 │ │  DNS→.101 │
   └───────────┘ └───────────┘ └───────────┘
```

**Fluxo de DNS:**
1. Dispositivo pede IP via DHCP ao D-Link
2. D-Link entrega IP + `DNS = 192.168.0.101` (Pi-hole)
3. Dispositivo consulta Pi-hole para todo hostname
4. Pi-hole: se domínio está em blocklist → responde `0.0.0.0` (bloqueado)
5. Se não está → encaminha para upstream (`1.1.1.1`, `1.0.0.1`)

---

## Componentes

| Componente | Detalhe |
|---|---|
| **Pi-hole** | v6.5 (Core), Web v6.6.1, FTL v6.8 — `192.168.0.101`, porta 53 |
| **Roteador** | D-Link DIR-809 (A2, firmware 1.07) — NAT duplo, DHCP-WAN |
| **Rede** | `192.168.0.0/24` (Quarto-wifi), gateway `192.168.0.1` |
| **Upstream DNS** | `1.1.1.1`, `1.0.0.1` (Cloudflare) — só quando não bloqueado |
| **DNS Secundário (D-Link)** | `0.0.0.0` (inválido de propósito — ver seção DoH abaixo) |

---

## Passo 1 — Instalar Pi-hole

### Instalação interativa (com terminal)

```bash
# No Raspberry Pi (ou dispositivo Linux)
curl -sSL https://install.pi-hole.net | bash
```

Durante a instalação:
- Interface: `wlan0` (ou a interface de rede do Pi)
- Upstream: **Cloudflare** (ou Google)
- Lists: escolha as que quiser (vamos substituir depois)
- Web admin: sim
- Query logging: sim
- **Admin password:** anote (usaremos para API/UI)

### Instalação não-interativa (sem TTY — como foi feito no Zero 2)

O instalador v6 usa `dialog` e **falha com "Error opening terminal"**
em SSH sem TTY. Truque: semear `setupVars.conf` (torna
`fresh_install=false` → pula todos os diálogos) e rodar `--unattended`:

```bash
sudo mkdir -p /etc/pihole
sudo tee /etc/pihole/setupVars.conf > /dev/null << 'EOF'
PIHOLE_INTERFACE=wlan0
IPV4_ADDRESS=192.168.0.101/24
QUERY_LOGGING=true
INSTALL_WEB_SERVER=true
INSTALL_WEB_INTERFACE=true
LIGHTTPD_ENABLED=true
BLOCKING_ENABLED=true
DNSMASQ_LISTENING=single
PIHOLE_DNS_1=1.1.1.1
PIHOLE_DNS_2=1.0.0.1
EOF

curl -sSL https://install.pi-hole.net -o /tmp/basic-install.sh
sudo bash /tmp/basic-install.sh --unattended
```

### Configurações pós-instalação

```bash
# Senha do admin (não-interativo: repete 2x)
printf 'NOVA_SENHA\nNOVA_SENHA\n' | sudo pihole setpassword

# Verificar
pihole status
```

> **Nota (v6):** `DNSMASQ_LISTENING=single` no setupVars vira
> `listeningMode=SINGLE` (escuta só em `wlan0`) — suficiente, pois todos
> os clientes estão na mesma interface. `pihole --setmode ALL` só é
> necessário se houver outras interfaces de rede.

Web admin: `http://192.168.0.101/admin`

### Credenciais

| Campo | Valor |
|---|---|
| URL | `http://192.168.0.101/admin` |
| Login | `admin` |
| Senha | definida na instalação |

> Para trocar: `sudo pihole setpassword`

---

## Passo 2 — Configurar o D-Link DIR-809

O firmware do DIR-809 **não tem campo de DNS no DHCP** e o DNS relay,
quando ligado, anuncia o IP do próprio roteador. A solução é:

1. **DNS relay → DESABILITADO**
2. **DNS WAN primário → IP do Pi-hole** (192.168.0.101)
3. **DNS WAN secundário → `0.0.0.0`** (por quê? ver seção DoH)
4. **Reserva DHCP → Pi-hole sempre em 192.168.0.101**

### Acesso ao painel

- URL: `http://192.168.0.1` (só acessível pela rede LAN, não pela WAN)
- Login: `admin` / senha **em branco** (padrão de fábrica)
- **Recomendação:** troque a senha

### Configuração via UI (recomendado — POST direto não persiste)

O firmware tem validação estranha e `RemoteDisconnected` em POSTs
diretos. Use a UI real (browser ou Playwright):

1. Login via `http://192.168.0.1`
2. **WAN (Basic → Internet):**
   - Servidor DNS primário: `192.168.0.101`
   - Servidor DNS secundário: **vazio**
3. **LAN (Basic → Network):**
   - DNS Relay: **Disable** (desabilitado)
4. **Reserva DHCP (Advanced → Network → DHCP Reservation):**
   - Hostname: `pi`
   - IP: `192.168.0.101`
   - MAC: `2c:cf:67:1c:c0:f3` (ou o MAC do seu Pi)
   - Expiração: **Nunca** (Never)
5. Salvar e **reiniciar** o roteador

### Verificar após reboot

```bash
# No Mac/dispositivo conectado ao Quarto-wifi
ipconfig getpacket en0 | grep domain_name_server
# Deve retornar: domain_name_server = 192.168.0.101
```

---

## Passo 3 — Configurar Blocklists no Pi-hole

Acesse `http://192.168.0.101/admin` → **Adlists** e adicione estas listas
(todas dinâmicas — atualizam automaticamente via cron):

| Lista | URL | Tipo |
|---|---|---|
| StevenBlack hosts | `https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts` | hosts |
| OISD full | `https://big.oisd.nl` | ABP/domains |
| AdGuard DNS filter | `https://adguardteam.github.io/AdGuardSDNSFilter/Filters/filter.txt` | ABP |
| AdAway | `https://adaway.org/hosts.txt` | hosts |
| 1hosts Lite | `https://raw.githubusercontent.com/badmojr/1Hosts/master/Lite/hosts.txt` | hosts |
| notracking | `https://raw.githubusercontent.com/notracking/hosts-blocklists/master/hostnames.txt` | hosts |
| Dandelion Sprout Anti-Malware | `https://raw.githubusercontent.com/DandelionSprout/adfilt/master/Alternate%20versions%20Anti-Malware%20List/AntiMalwareHosts.txt` | hosts |
| MVPS hosts | `http://winhelp2002.mvps.org/hosts.txt` | hosts |
| Dan Pollock | `https://someonewhocares.org/hosts/zero/hosts` | hosts |
| pgl.yoyo adservers | `https://pgl.yoyo.org/adservers/serverlist.php?hostformat=hosts&showintro=1&mimetype=plaintext` | hosts |
| HaGeZi Pro | `https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/pro.txt` | ABP |
| Phishing Army | `https://phishing.army/download/phishing-army-hosts.txt` | hosts |

> **~1.12M domínios únicos** após gravity update.
>
> Além disso, `connect.facebook.net` (pixel da Meta) não vem em nenhuma
> das 12 listas — foi adicionado como **deny exato** via API (ver abaixo).

### Deny exatos (fora das lists)

```bash
# connect.facebook.net (Meta pixel) — POST /api/domains/{type}/{kind}
curl -s -X POST "http://192.168.0.101/api/domains/deny/exact" \
  -H "sid: $SID" -H "Content-Type: application/json" \
  -d '{"domain":"connect.facebook.net","comment":"Meta pixel tracker"}'
```

### Atualizar gravity (após adicionar listas)

```bash
sudo pihole updateGravity
```

Saída esperada:
```
[i] Number of gravity domains: 1600345 (1119371 unique domains)
```

### Atualização automática

O Pi-hole já tem cron configurado (hora aleatória, definida na instalação):
```cron
# /etc/cron.d/pihole (exemplo real)
6 4 * * 7 pihole PATH="$PATH:/usr/sbin:/usr/local/bin/" pihole updateGravity
```
Todo domingo (04:06 no install atual) baixa as versões mais recentes das
listas. Como as URLs são dinâmicas (GitHub raw, OISD, etc.), sempre
recebe o conteúdo atualizado.

---

## Passo 4 — Por que o DNS secundário está VAZIO (DoH Bypass)

### O problema

Quando `1.1.1.1` (Cloudflare) está na lista de resolvers do macOS (via
lease DHCP), o Chrome em modo **"Usar DNS seguro"** (Secure DNS, auto)
detecta o resolver do sistema e faz **upgrade automático para DoH usando
Cloudflare** — ignorando o Pi-hole completamente.

Sintoma: anúncios carregam normalmente no Chrome, mas `dig`/`curl`
mostram bloqueio. O Chrome vai direto para o Cloudflare DoH, que não
tem as blocklists.

### A solução

Remover o resolver público do lease DHCP. Como o firmware do DIR-809
tem um campo de secundário livre, usamos **`0.0.0.0`** como
secundário:

- `domain_name_server = {192.168.0.101}` (o `0.0.0.0` é descartado
  como inválido — o lease nem sai)
- Chrome não consegue fazer upgrade de DoH (não há resolver público
  para detectar)
- Todas as consultas passam pelo Pi-hole, sem exceção

> Histórico: durante uma pane do Pi-hole adicionamos `8.8.8.8` como
> secundário para restaurar a internet. Com o Pi-hole de volta, o
> `8.8.8.8` foi **removido** (substituído por `0.0.0.0`) para eliminar
> o risco de DoH bypass.

### Trade-off

Se o Pi-hole cair, **não há fallback DNS** — resolução de nomes para.
Alternativa: desativar "Usar DNS seguro" no Chrome
(`chrome://settings/security`) e manter `1.1.1.1` como secundário.

---

## Passo 5 — Verificar

### Teste DNS

```bash
# Deve retornar 0.0.0.0
dig @192.168.0.101 doubleclick.net A
dig @192.168.0.101 googleadservices.com A
dig @192.168.0.101 pagead2.googlesyndication.com A
dig @192.168.0.101 an.facebook.com A

# Deve retornar IP real (não bloqueado)
dig @192.168.0.101 google.com A
dig @192.168.0.101 facebook.com A
```

### Teste no navegador

Acesse `chrome://settings/security` → "Usar DNS seguro" deve estar em
"Padrão do sistema" (não "Cloudflare" ou outro). Se aparecer um
resolvedor seguro, o bypass de DoH pode estar ativo.

### Estatísticas

Web admin → Dashboard mostra total de consultas, bloqueadas, clientes
ativos, etc. API:

```bash
# Autenticar
SID=$(curl -s -X POST http://192.168.0.101/api/auth \
  -d '{"password":"SUA_SENHA"}' | jq -r .session.sid)

# Estatísticas
curl -s -H "sid: $SID" http://192.168.0.101/api/stats/summary | jq .
```

---

## DNS Local — Nomes em vez de IPs

Como o Pi-hole é o DNS de toda a rede, registros locais (A records)
funcionam automaticamente para **todos os dispositivos**, sem instalar
nada.

### Configuração atual

| Nome | IP | Acesso |
|---|---|---|
| `pi.local` | `192.168.0.101` | Pi-hole (admin em `http://pi.local/admin`) |
| `pihole.local` | `192.168.0.101` | Pi-hole (alias) |
| `router.local` | `192.168.0.1` | D-Link DIR-809 |

### Como adicionar novos nomes

**Via UI:** Local DNS → DNS Records → adicionar `IP` + `domínio`

**Via API:**
```bash
SID=$(curl -s -X POST http://192.168.0.101/api/auth \
  -d '{"password":"SUA_SENHA"}' | jq -r .session.sid)

curl -s -X PATCH -H "sid: $SID" -H "Content-Type: application/json" \
  http://192.168.0.101/api/config \
  -d '{"config":{"dns":{"hosts":[
    "192.168.0.101 pi.local",
    "192.168.0.1 router.local",
    "192.168.0.102 nas.local"
  ]}}}'
```

> **Formato:** `"IP hostname"` (um por item do array). A alteração é
> aplicada imediatamente, sem reiniciar.

### Teste

```bash
dig +short pi.local A        # → 192.168.0.101
host router.local            # → 192.168.0.1
curl http://pi.local/admin/  # → 302 (redirect p/ login)
```

### Aviso sobre `.local`

`.local` é o domínio reservado para **mDNS/Bonjour** (RFC 6762). Funciona
bem com o Pi-hole em unicast DNS, mas em redes com muitos dispositivos
Apple pode haver conflito (o macOS consulta mDNS para `.local`). Se
notar lentidão ou falha de resolução, troque para `.lan`, `.home` ou
`.internal`:

```bash
# Exemplo: trocar para .lan
curl -s -X PATCH -H "sid: $SID" -H "Content-Type: application/json" \
  http://192.168.0.101/api/config \
  -d '{"config":{"dns":{"hosts":["192.168.0.101 pi.lan"]}}}'
```

---

## Limitações Conhecidas

### YouTube — anúncios de vídeo e in-feed

O YouTube entrega anúncios dos **mesmos domínios** dos vídeos
(`youtube.com`, `googlevideo.com`, `ytimg.com`). O DNS não distingue
"anúncio" de "conteúdo" quando estão no mesmo hostname. Bloquear esses
domínios derrubaria o YouTube inteiro.

**Soluções (por dispositivo, não via rede):**
- uBlock Origin (navegador)
- NewPipe/ReVanced (Android), SmartTube (TV Android)
- YouTube Premium

### Anúncios first-party em sites

Alguns sites usam anúncios em domínio próprio (ex: Globo usa
`ads-library.globo.com`). Esses domínios específicos não estão em
listas genéricas. No entanto, os anúncios de terceiros (DoubleClick,
Google Ads) são bloqueados e frequentemente deixam o espaço vazio.

### Sem fallback DNS

Como o secundário está vazio, se o Pi-hole cair a rede fica sem
resolução de nomes até ele voltar. Para produção, considere monitorar
o Pi-hole ou manter um secundário (aceitando o risco de DoH bypass).

---

## Reprodução Rápida (checklist)

- [ ] Instalar Pi-hole (`curl -sSL https://install.pi-hole.net | bash`)
      — sem TTY, usar método `setupVars.conf` + `--unattended` (Passo 1)
- [ ] Definir password do admin (`sudo pihole setpassword`)
- [ ] Adicionar as 12 blocklists (seção Passo 3)
- [ ] Deny exato: `connect.facebook.net` (seção Passo 3)
- [ ] `sudo pihole updateGravity`
- [ ] Registros locais: `pi.local`, `pihole.local`, `router.local` (seção DNS Local)
- [ ] Configurar D-Link: DNS relay OFF, DNS WAN = 192.168.0.101, secundário `0.0.0.0`
- [ ] Reserva DHCP para Pi em 192.168.0.101 (`2c:cf:67:1c:c0:f3`)
- [ ] Reiniciar D-Link
- [ ] Verificar lease DHCP: `domain_name_server = {192.168.0.101}`
- [ ] Testar `dig @192.168.0.101 doubleclick.net A` → 0.0.0.0
- [ ] Testar `dig @192.168.0.101 pi.local A` → 192.168.0.101
- [ ] Verificar Chrome: Secure DNS = "Padrão do sistema"

---

## Ocorrência: Pane e Recuperação (19/10/2026)

### O que houve

1. Durante um `updateGravity` (escrita grande no SQLite), a energia
   foi cortada no meio da gravação
2. O NetworkManager estava **salvando o perfil Wi-Fi** (escrita atômica
   em arquivo temporário `.nmconnection.XXXXXX`) — o corte deixou o
   arquivo final **sem conter nada** e só o temp vazio
3. Efeito: Pi ligado, LED verde aceso, **mas `wlan0` sem perfil** →
   nunca mais subiu no Wi-Fi → sem DHCP lease → sem Pi-hole →
   a rede inteira ficou sem DNS
4. O fs ext4 ficou com erros menores (bitmap/contadores de blocos —
   `e2fsck -nf` acusa, mas não impede boot)

### Por que o Pi "sumiu" da rede

- D-Link: Pi ausente da tabela DHCP **e** da lista de clientes
  wireless (só 3 clientes) → confirma que o problema era o **Wi-Fi
  do Pi**, não o Pi-hole em si
- Sem cabo de rede disponível (Zero 2 W só tem Wi-Fi) → diagnóstico
  via **SD card no Mac** (debugfs read-only)

### Recuperação executada

1. **Formatação da SD + imagem nova** (Raspberry Pi OS 32-bit,
   "Zero 2" selecionado no Imager, SSH habilitado, user `art`)
2. IP `.101` **voltou automaticamente** — a reserva DHCP da D-Link
   é por MAC (`2c:cf:67:1c:c0:f3`), não muda com reinstall
3. Pi-hole instalado via `setupVars.conf` + `--unattended` (Passo 1)
4. Password, 12 blocklists, gravity, deny exato e registros locais
   refeitos (tudo documentado neste README)
5. DNS secundário da D-Link: `8.8.8.8` (workaround da pane) →
   **`0.0.0.0`** (elimina risco de DoH bypass)

### Lições

- **`sync` antes de desligar**: nunca cortar energia durante
  `pihole -g` (domingo 04:06 — cuidado com apagões nesse horário)
- O netplan/NM do Raspberry Pi OS escreve o perfil Wi-Fi em
  `/etc/NetworkManager/system-connections/` — se o Pi voltar sem
  Wi-Fi após queda de energia, checar esse diretório na SD
- Reserva DHCP por MAC no roteador torna o `.101` sobrevivente a
  qualquer reinstall

---

## Arquivos neste repositório

| Arquivo | Descrição |
|---|---|
| `README.md` | Este documento — arquitetura completa |
| `dir-809-info.md` | Detalhes técnicos do D-Link DIR-809 |
| `ge-globo-ads-blocked.png` | Screenshot de teste (ads bloqueados no ge.globo.com) |

> Logs de sessão e credenciais **não** são versionados
> (`.gitignore`: `*.log`, `session-*.md`, `.playwright-mcp/`).

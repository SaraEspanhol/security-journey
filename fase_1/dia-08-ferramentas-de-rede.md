# Dia 08 — Ferramentas de rede na prática

**Fase 1 — Fundamentos de Redes**
Conteúdo do plano: Dias 13-14 (consolidação)
Ferramentas: `ping`, `traceroute`, `ifconfig`, `netstat -rn`, Wireshark

---

## 1. `ping` — mecanismo

Não fala TCP nem UDP. Fala **ICMP** (camada de rede, mesma do IP).
- **Echo Request** (Type 8): "você está aí?"
- **Echo Reply** (Type 0): "estou, devolvo."

Dispara e mede o **RTT** (round-trip time). Sem handshake, sem conexão.

### Leituras de analista
- **RTT**: distância/qualidade do caminho. `<1 ms` = LAN; `~15 ms` = mesma região; `~180 ms` = outro lado do mundo ou caminho ruim.
- **Ler a distribuição, não só a média**: um primeiro pacote alto (ARP, cache de rota frio) infla a média. `stddev` alto relativo à latência denuncia outlier. RTT de regime ≠ média isolada.
- **TTL da resposta = saltos no caminho de VOLTA** (não de ida). O caminho de ida pode ter nº diferente de saltos → *roteamento assimétrico*. TTL inicial típico: Linux/macOS = 64, Windows = 128.
  - Ex.: resposta com `ttl=57` → `64 - 57 = 7 saltos` no retorno; SO provável Unix-like (hedgear, é sinal, não prova).

### macOS (BSD) vs Linux (GNU)
- `ping -c 4` → 4 pacotes e para.
- Pegadinha: no **macOS** `ping -t` é *timeout*; no **Linux** `ping -t` é *TTL*. Mesma flag, comportamento diferente.

### Relevância SOC
ICMP é recon (`nmap -sn` = ping em escala) e vetor de exfiltração (**ICMP tunneling** — dados no payload do Echo, passa por firewall que só olha TCP/UDP).

---

## 2. `traceroute` — o abuso do TTL

Não "vê" o caminho. **Abusa do TTL** para forçar cada roteador a se identificar.

1. Envia pacote com **TTL=1** → primeiro roteador decrementa para 0 → descarta e devolve **ICMP Time Exceeded** (Type 11), *do IP dele*. Salto 1 revelado.
2. TTL=2 → passa o primeiro, morre no segundo → Time Exceeded. Salto 2.
3. Incrementa até chegar ao destino.

### Como sabe que chegou ao fim
- **UDP** (padrão macOS/Linux): manda para porta altíssima improvável → destino responde **ICMP Port Unreachable** (Type 3, Code 3).
- **ICMP** (`-I`): Echo Reply, igual ao ping.
- **TCP** (`-T -p 443`, precisa `sudo`): SYN para 443; útil quando UDP/ICMP são filtrados.

### `* * *`
Roteador **não respondeu** (política de não gerar ICMP, rate-limit, filtro). O salto seguinte aparece normal porque **encaminhar ≠ responder**: o roteador mudo continua repassando pacotes, só não fala de si mesmo.

### Leitura de evidência (da captura real)
- **`100.64.0.0/10` = CGNAT** (RFC 6598). 4º bloco especial, além dos privados (`10/8`, `172.16/12`, `192.168/16`). Provedor NATeia vários clientes juntos → atribuição por IP público fica fraca (precisa porta origem + timestamp).
- **Reverse-DNS do salto revela o provedor** (ex.: `starlinkisp.net`). Técnica de recon padrão.
- **Múltiplos IPs no mesmo salto = ECMP** (balanceamento de carga, caminhos paralelos de igual custo). NÃO é instabilidade.
- "Destino a N saltos por *este* caminho" — com ECMP, o caminho nem é único.

---

## 3. `ifconfig` e tabela de rotas

`ifconfig en0` — campos:
- `inet` → IP privado
- `netmask 0xffffff00` → máscara (`/24` = `255.255.255.0`)
- `broadcast` → difusão da sub-rede
- `ether` → MAC (camada 2)
- `status: active` → link de pé

Gateway **não** está no `ifconfig` → está na tabela de rotas:
```
netstat -rn | grep default
```
Linha `default` → gateway = roteador de casa (porta de saída da LAN).

### Flags da rota (`UGScg`)
- **U** = Up (usável)
- **G** = Gateway (exige encaminhamento por intermediário)
- **S** = Static
- minúsculas (cloning/escopo): confirmar em `man netstat` — não inventar.

### A parede do `ip`
`ip a` → `command not found`. `ip`/`ss` são do **iproute2 (Linux)**; macOS é **BSD userland**.
Equivalências para levar à Fase 2:
| Linux (iproute2) | macOS (BSD) |
|------------------|-------------|
| `ip a`           | `ifconfig`  |
| `ip r`           | `netstat -rn` |
| `ss -tlnp`       | `lsof -iTCP -sTCP:LISTEN` |

### Dual-stack (achado do dia)
Interface tem vários `inet6`:
- `fe80::...` → **link-local** (só vale no segmento local)
- `fd..::...` → **ULA** (`fc00::/7`) = equivalente IPv6 do IP privado
- `2xxx:...` → **global unicast** = IPv6 público, roteável. O marcado `temporary` é *privacy extension* (RFC 4941), o SO rotaciona sozinho.

**Ponto SOC crítico**: IPv4 atrás de duplo NAT (casa + CGNAT) ≠ IPv6 global exposto. Regra de firewall só para v4 deixa o v6 escancarado. `iptables` cheio + `ip6tables` vazio = host exposto. "Protegido por NAT" só vale no v4.

`utun0..N` = túneis userland (VPN, iCloud Private Relay). Conhecer as interfaces legítimas da própria máquina é o que permite notar uma que não deveria estar ali (túnel = vetor de exfiltração).

---

## 4. Wireshark — captura e leitura

### Capture filter vs Display filter (a confusão nº 1 da ferramenta)
- **Capture filter**: sintaxe **BPF**, aplicado *antes* de gravar. Só entra o que casa. Ex.: `host 1.1.1.1`. Fica na tela inicial, perto da lista de interfaces.
- **Display filter**: dialeto **Wireshark**, aplicado *depois*, sobre o que já foi capturado. Esconde, não descarta. Ex.: `ip.addr == 1.1.1.1`, `tcp.stream == N`. Barra larga no topo da janela.

Regra: **capture filter para não capturar lixo; display filter para achar agulha no que sobrou.**
Erro comum: digitar `host 1.1.1.1` (BPF) na barra de display filter → vermelho (display não conhece `host`; quer `ip.addr ==`).

### Três painéis
Lista de pacotes (topo) → detalhe/encapsulamento (meio: Frame → Ethernet L2 → IP L3 → TCP L4 → payload) → bytes crus/hex (baixo). O painel do meio é a cebola das camadas desmontada.

### Three-way handshake na tela
```
SYN:      cliente:60391 → 1.1.1.1:443  [SYN]      Seq=0
SYN-ACK:  1.1.1.1:443 → cliente:60391  [SYN, ACK] Seq=0 Ack=1
ACK:      cliente:60391 → 1.1.1.1:443  [ACK]      Seq=1 Ack=1
```

Mecanismo por baixo dos números:
- **SYN consome 1 no espaço de sequência mesmo com `Len=0`** → por isso SYN-ACK responde `Ack=1` e não `Ack=0`. FIN faz o mesmo. (SYN e FIN cada um gastam 1 número de sequência.)
- **Window scaling negociado só no handshake**: SYN anuncia `WS`; depois `Win` cresce além do 65535 inicial. Se a captura perder o SYN/SYN-ACK, o Wireshark não sabe o fator de escala e mostra tamanhos de janela errados → por isso analista quer o handshake dentro da captura.
- **Four-way teardown** espelha a abertura: `FIN,ACK` → `ACK` → `FIN,ACK` → `ACK`. Abre com SYN, fecha com FIN.

### Follow TCP Stream — o enterro da pendência do Dia 2
Botão direito no pacote → Follow → TCP Stream. Wireshark gera sozinho `tcp.stream == N`.
- `tcp.stream` **não é campo do pacote** — não existe no cabeçalho TCP. É etiqueta que o Wireshark inventa, agrupando pacotes pelo **four-tuple** (IP orig : porta orig ↔ IP dest : porta dest). É a conta do Dia 5 virada botão.
- O número N = contagem de conexões TCP na captura (0-indexado). N=16 → havia ≥17 streams. Mesmo com capture filter por endereço, várias conexões distintas ao mesmo IP = vários streams.
- **Por que resolve o Dia 2**: no Dia 2 a tentativa foi garimpar o stream manualmente no ruído da interface inteira. Faltava estratégia, não técnica: capture filter corta na origem + Follow calcula o four-tuple e entrega o stream pronto.

---

## 5. Triagem SOC com `netstat`

```
netstat -an -p tcp | grep ESTABLISHED
```
Cada linha = um **four-tuple** único. Duas conexões ao mesmo IP:porta destino coexistem porque as **portas de origem diferem** (mesma lógica do `tcp.stream`). O mesmo objeto do Dia 5 visto por três janelas: netstat, lsof, Wireshark.

### Método de triagem
Separar **o que sei** (ESTABLISHED, porta, associação típica) de **o que NÃO sei** (qual aplicação abriu, qual serviço do outro lado).

**Branch 2**: porta desconhecida → identificar o processo com `lsof` **antes** de assumir benigno. Não inventar o serviço a partir do número da porta — puxar o processo (`lsof -iTCP:<porta_origem>`).

**Armadilha da porta conhecida**: 443 não é atestado de inocência. 443 aberta só diz que *algo fala TLS para fora*, não *o quê*. C2/malware moderno usa 443 de propósito para se esconder no tráfego web. A pergunta "qual processo abriu?" vale **inclusive** na 443. Porta conhecida não dispensa a branch 2 — só torna a preguiça mais tentadora.

---

## Acertos nomeados
- Leitura de saída com precisão de campo (Ex 1, 2, 3).
- Mecanismo sobre fórmula (TTL do traceroute, encaminhar≠responder).
- Hedge de inferência correto ("provavelmente Unix/Linux").
- **Branch 2 espontânea** no Ex 5 (primeira vez reflexa).
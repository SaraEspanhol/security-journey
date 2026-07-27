# Dia 6 — DNS a fundo

**Fase 1 — Fundamentos de Redes**
**Conteúdo do plano original: Dias 9-10 (adiantado)**
**Erro do diagnóstico fechado:** o caminho completo ao carregar um site (DNS → TCP → TLS → HTTP)

---

## 1. O problema que o DNS resolve

Ninguém digita IP no navegador. O DNS (1983) resolve a tradução nome→IP com três decisões de projeto:

- **Hierarquia**: espaço de nomes como árvore invertida — cada nível só sabe apontar para o próximo.
- **Delegação**: quem administra `.com` não sabe o IP de `google.com`; sabe *quem sabe*.
- **Cache**: resposta guardada por um tempo definido pelo dono do domínio (TTL).

Todo comportamento anômalo de DNS em triagem de SOC é uma dessas três coisas dando errado ou sendo abusada.

## 2. Árvore de nomes

```
.                      → root
br.                    → TLD
com.br.                → segundo nível
exemplo.com.br.        → domínio registrado (zona)
www.exemplo.com.br.    → host dentro da zona
```

- **FQDN**: nome com ponto final, absoluto. Sem o ponto, pode virar relativo e ganhar sufixo de busca local (`search domain`) — vazamento de consulta.
- **Zona**: unidade de administração. O servidor que a hospeda é o **autoritativo** — não tem cache, tem o dado original.

## 3. Os três atores

- **Stub resolver**: biblioteca do SO. Manda a pergunta pro recursivo configurado e espera resposta pronta.
- **Recursive resolver**: roteador / ISP / `1.1.1.1` / `8.8.8.8`. Faz o trabalho pesado, sobe a árvore. **É aqui que vive o cache que importa.**
- **Authoritative servers**: root, TLD, zona. Cada um responde só o que é da sua alçada.

**Consulta recursiva** (stub → recursivo): "me dê a resposta final."
**Consulta iterativa** (recursivo → árvore): "me diga o que você sabe, mesmo que seja só o próximo passo."

Flags no `dig` que provam isso:
- `rd` — recursion desired (pedido pelo cliente)
- `ra` — recursion available (o servidor oferece)
- `aa` — authoritative answer (quem respondeu é dono da zona)

## 4. Cache e TTL

TTL = contrato em segundos: "pode guardar por N segundos, depois pergunte de novo". Definido pelo **dono da zona**, decrementado pelo **cache do recursivo**.

- `dig` não passa pelo cache do macOS — fala direto com o resolver configurado.
- O autoritativo não decrementa nada: ele *emite* o TTL cheio sempre, porque não tem estado, tem o dado original.
- Cache do macOS (se precisar): `sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder`

## 5. Tipos de registro

| Tipo | Função | Observação |
|------|--------|------------|
| A | nome → IPv4 | caso base |
| AAAA | nome → IPv6 | "quad-A" |
| CNAME | nome → outro nome (alias) | não pode coexistir com outro registro no mesmo nome; não pode estar no ápice da zona |
| MX | domínio → nome do servidor de e-mail | tem prioridade (menor = preferido); aponta para **nome**, nunca IP — exige segunda resolução |
| NS | zona → servidores autoritativos | registro da delegação |
| TXT | texto livre | SPF/DKIM/DMARC, verificações de propriedade |
| PTR | IP → nome (reverso) | zona `in-addr.arpa`, octetos invertidos |
| SOA | metadados da zona | servidor primário, serial, timers |
| HTTPS (tipo 65) | parâmetros de conexão HTTPS | ALPN (HTTP/2, HTTP/3-QUIC), permite pular TCP clássico e ir direto pra QUIC |

## 6. Transporte: UDP com fallback para TCP

DNS usa **porta 53**, **UDP por padrão** — a consulta cabe num pacote, reenviar custa menos que abrir handshake pra 100 bytes.

Cai para **TCP 53** em dois casos:
- **Resposta grande**: acima do limite negociado (EDNS0, tipicamente ~1232-4096 bytes hoje; 512 no clássico) → flag `TC` (truncated) → resolver refaz em TCP.
- **AXFR** (transferência de zona): sempre TCP, cópia integral entre autoritativos. Servidor que aceita AXFR de qualquer um vaza a infraestrutura interna inteira.

Também existem **DoT** (porta 853) e **DoH** (porta 443, indistinguível de tráfego web) — criptografam a consulta. Bom pra privacidade, ruim pra visibilidade de SOC.

## 7. O caminho completo ao carregar um site (erro do diagnóstico)

```
1. Resolução DNS   → stub checa cache/hosts, senão UDP:53 pro recursivo. Nada de TCP ainda.
2. Handshake TCP   → porta efêmera de origem, SYN/SYN-ACK/ACK pro IP:443. Canal confiável, zero cripto.
3. Handshake TLS   → ClientHello (com SNI em texto claro) → ServerHello → certificado → 
                      validação de cadeia → troca de chaves. Canal cifrado a partir daqui.
4. HTTP            → GET dentro do túnel TLS. Resposta. Repete o ciclo pra cada novo domínio.
```

**Correção em cima do modelo clássico**: se o registro HTTPS (tipo 65) anunciar suporte a QUIC, o passo 2/3 pode ser substituído por handshake QUIC sobre UDP direto — o cliente só sabe que pode pular porque perguntou no passo 1.

Isolamento por camada = método de trabalho de SOC: `dig` testa passo 1, `nc -zv host 443` testa passo 2, `openssl s_client` testa passo 3, `curl -v` testa passo 4.

**Nomes trafegam em claro nos passos 1 e 3** (query DNS e SNI) mesmo com HTTPS — por isso o log de DNS é a fonte mais rica de um SOC mesmo num mundo todo criptografado.

## 8. DNS na ótica de SOC

- **DNS tunneling**: dados codificados nos nomes/registros consultados. Assinaturas: nomes longos, alta entropia, volume alto num único domínio, uso pesado de TXT/NULL.
- **DGA**: malware gera centenas de domínios pseudoaleatórios/dia, tenta todos. Assinatura: **rajada de NXDOMAIN** do mesmo host.
- **Fast flux**: mesmo nome trocando de IP com TTL baixíssimo, resiste a takedown.
- **Typosquatting**: `micros0ft.com`. Base de phishing.
- **Cache poisoning**: injetar resposta falsa no cache. Mitigação estrutural: **DNSSEC**.
- **DoH bloqueado**: empresas bloqueiam resolvers DoH conhecidos pra não perder telemetria.
- **DNS sinkhole** (defensivo): resolver interno responde domínios maliciosos com IP controlado — mata a comunicação e lista as máquinas infectadas.

### DNSSEC observado via `+trace`

- **RRSIG**: assinatura de um conjunto de registros — garante autenticidade e integridade daquele conjunto.
- **DS** (Delegation Signer): publicado **pela zona pai sobre a zona filha**, atesta a chave pública do próximo nível. Root assina `.com`, `.com` assina `example.com`. Corrente de confiança elo por elo.
- A zona folha (ex. `example.com` respondendo por si) tem `RRSIG A` mas **não** tem DS — porque DS existe só para validar delegação a um próximo nível, e a folha não delega pra ninguém.
- Flag `ad` (Authenticated Data) indica que a resposta foi validada via DNSSEC — não aparece em `+trace` porque `+trace` não valida, só mostra a cadeia crua.
- DNSSEC dá **autenticidade e integridade**, não **confidencialidade** — assina, não cifra.

---

## Exercícios e correção — resumo

### Exercício 1 — Ambiente e stub resolver
`scutil --dns` lido corretamente: `192.168.1.1` como recursivo primário, `fd91:...` (ULA, mesmo roteador via IPv6), interface `en0`. Ponto reforçado: distinguir "quem eu consulto" de "quem resolve de fato lá atrás".

### Exercício 2 — Anatomia da resposta e cache
- Seções QUESTION/ANSWER/ADDITIONAL lidas certo, incluindo o registro OPT (EDNS) contado como adicional.
- Flags `qr rd ra` explicadas certo; ausência de `aa` corretamente lida como "não é autoritativo".
- **TTL 182→83 em 84s**: anomalia não investigada na primeira tentativa (assumida como "ajuste do resolvedor"). 
- Consulta ao NS direto (`elliott.ns.cloudflare.com`): TTL cheio (300), flags `qr aa` (sem `rd`/`ra`) — resposta de fonte, não de cache.

### Exercício 3 — Tipos de registro
A, AAAA, MX (com prioridade), NS, TXT, CNAME real (`www.microsoft.com` → `edgekey.net` → `akamaiedge.net` → A final) obtidos e lidos com output cru.
- CNAME sozinho não resolve para IP — resolver só segue a cadeia quando o tipo pedido é A/AAAA.
- TTL do IP final da cadeia Akamai: 20s (CDN — precisa redirecionar rápido) vs. 300s do `example.com` estático (nada pra rebalancear). Trade-off geral: TTL alto = menos carga, propagação lenta; TTL baixo = controle fino, mais consultas.
- MX aponta para nome, não IP: permite trocar infraestrutura sem alterar o registro; remetente precisa resolver o nome (A/AAAA) antes de abrir TCP:25 — duas resoluções DNS antes de qualquer SMTP.
- TXT do `google.com`: SPF encontrado em meio a 15 linhas via leitura completa (não resumo); uma linha isolada (`Z29vZ2xl`) decodificada via base64 como exercício de não descartar dado que destoa do padrão.

### Exercício 4 — `+trace`
Quatro rodadas lidas certo: `1.1.1.1` → root (`k.root-servers.net`) → TLD (`j.gtld-servers.net`) → autoritativo (`hera.ns.cloudflare.com`). Ponto de virada identificado certo: registros NS = delegação, registros A = resposta final. DNSSEC (RRSIG/DS) presente em cada rodada, lido e explicado no fechamento.

### Exercício 5 — DNS no fio e triagem

**Parte a**: captura real com tráfego de fundo (Spotify, Google, IFSP). Fluxo isolado corretamente via porta efêmera + transaction ID — mesma técnica de isolamento de fluxo específico do Wireshark (Dia 2), aplicado aqui com sucesso em contexto DNS/UDP.
Observado: toda consulta sai em trio A + AAAA + **HTTPS (tipo 65)** em paralelo — ver seção 5 acima.
`dig +tcp` capturado em paralelo com tcpdump.

**Parte b** — triagem do log `corp-update.net`:
- Padrão identificado: rajada de `NXDOMAIN` com nomes pseudoaleatórios (assinatura de **DGA** — descoberta do C2) seguida de `TXT`/`NOERROR` com strings Base64 (assinatura de **tunneling** — exfiltração), em sequência temporal consistente com descoberta→exfiltração.
- Hipótese concorrente (serviço legítimo) pesada contra o padrão de NXDOMAIN em rajada, que não é comportamento de app já configurado com nome correto.
- Próximo passo de coleta: identificar processo/PID gerando as consultas no host, binário de origem, se o domínio é serviço autorizado — cadeia PID → binário → legitimidade (ramo 2 da triagem, Dia 5).

---

## Padrão a vigiar (reforçado neste dia)

**Tendência a resumir ou descrever o resultado esperado no lugar de colar output real**, mais frequente em blocos repetitivos ou sob cansaço — distinto do "mira da resposta" original (Dia 4/5), porque aqui a teoria já estava sólida; o risco estava na entrega da evidência, não na compreensão do conceito.

Reforço positivo: quando a correção forçou parada e nova tentativa, a investigação subsequente foi sólida — teste de série de TTL, whois cruzado com whoami.akamai, decodificação de Base64 sem hesitar. O padrão não é falta de capacidade — é o impulso de entregar rápido competindo com o de observar primeiro.

---

## Conexão com vagas / entrevistas de SOC

- "Explique o que acontece quando você digita uma URL" é pergunta praticamente garantida em entrevista júnior — a cadeia DNS→TCP→TLS→HTTP com mecanismo (não slogan) é exatamente o que se espera.
- Leitura de log de resolver (NXDOMAIN em rajada, TXT anômalo) é trabalho real de N1 — reconhecer DGA e tunneling por padrão, não por assinatura de antivírus.
- DNSSEC, DoH/DoT e a perda de visibilidade que causam são temas correntes em discussões de arquitetura de segurança corporativa.
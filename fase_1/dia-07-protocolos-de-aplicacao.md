# Dia 7 — Protocolos de Aplicação: HTTP, HTTPS e TLS Handshake

Fase 1 — Fundamentos de Redes (conteúdo do plano: Dias 11-12)

## 1. HTTP — anatomia

Protocolo de aplicação, texto puro (HTTP/1.1), roda sobre TCP (porta 80).
Modelo request/response, stateless (memória entre requests vem de fora:
cookies, headers, sessão).

**Request:**
```
GET /index.html HTTP/1.1     ← request line: método, path, versão
Host: example.com            ← headers
User-Agent: curl/8.7.1
Accept: */*
                              ← linha em branco separa header de corpo
[corpo opcional — GET normalmente não tem; POST tem]
```

**Response:**
```
HTTP/1.1 200 OK               ← status line
Content-Type: text/html
Content-Length: 1256
Server: nginx

<html>...</html>              ← corpo
```

`Host` é obrigatório no HTTP/1.1: permite vários sites (virtual hosts) no
mesmo IP — o servidor decide qual responder olhando esse header. No HTTPS o
equivalente é o SNI, dentro do TLS (ver seção 3).

### Métodos
- **GET** — pedir recurso, não deve alterar nada no servidor.
- **POST** — enviar dados para processar (login, form, upload). Tem corpo.
- **HEAD** — igual GET mas só headers, sem corpo. Serve para checar
  metadados sem baixar conteúdo (`curl -I`).
- **PUT/DELETE/PATCH** — criar/substituir, apagar, alterar parcial (APIs REST).
- **OPTIONS** — pergunta métodos aceitos pelo recurso.
- **TRACE** — ecoa o request; quase sempre desabilitado (abusável).

Ângulo SOC: método inesperado para um recurso é sinal de triagem (ex.:
`DELETE`/`PUT` num endpoint que só deveria receber `GET`).

### Status codes — famílias
- **1xx** informativo — raro em log de análise.
- **2xx** sucesso — `200 OK`, `201 Created`, `204 No Content`.
- **3xx** redirecionamento — `301 Moved Permanently`, `302 Found`.
- **4xx** erro do cliente — `400`, `401 Unauthorized` (não autenticado),
  `403 Forbidden` (autenticado sem permissão), `404`, `429 Too Many Requests`.
- **5xx** erro do servidor — `500`, `502 Bad Gateway`, `503`.

**Padrões de status = assinatura de ataque em log:** rajada de `404` de um
mesmo IP em paths que não existem (`/admin`, `/.env`) = enumeração/scan.
Sequência de `401`/`403` num login = brute force/credential stuffing. Pico
de `500` pode ser exploração quebrando a aplicação. Lê-se a **forma** do
tráfego, não "o erro" isolado.

### Headers de segurança relevantes
Response: `Strict-Transport-Security` (HSTS, força HTTPS),
`Content-Security-Policy` (mitiga XSS), `X-Frame-Options` (anti-clickjacking),
`Set-Cookie` com `Secure`/`HttpOnly`. Ausência é achado de auditoria.
Request: `Authorization`, `Cookie`, `User-Agent` — onde credenciais e
fingerprint do cliente aparecem.

Banner de versão exposto (`Server: Apache/2.4.66`) é information disclosure
— facilita mapear CVE conhecido para a versão.

## 2. HTTPS = HTTP dentro de um túnel TLS

Não é protocolo diferente na camada de aplicação — é o mesmo HTTP, dentro de
um túnel TLS, porta 443. Pilha: TCP → TLS → HTTP.

## 3. TLS handshake

### TLS 1.2 (clássico), após o TCP three-way handshake:
1. **ClientHello** — versões TLS suportadas, cipher suites, client random,
   **SNI** (hostname pretendido, em claro).
2. **ServerHello** — servidor escolhe uma versão e uma cipher suite, manda
   server random.
3. **Certificate** — cadeia de certificados (site + intermediários até CA raiz).
4. **ServerKeyExchange + ServerHelloDone** — parâmetros de troca de chave
   efêmera (ECDHE).
5. Cliente valida o certificado (cadeia até CA confiável, validade, hostname
   bate com SNI) — **isto é o coração da autenticação**. Depois
   ClientKeyExchange: derivação do segredo de sessão.
6. Ambos derivam a mesma chave simétrica de sessão (nunca trafega pela rede).
7. **ChangeCipherSpec + Finished** dos dois lados — daqui em diante, tudo cifrado.

Assimétrica monta o cofre (troca segura da chave), simétrica usa o cofre
(cifra a conversa, rápida).

### O que TLS 1.3 mudou
- **1-RTT** em vez de 2 — `ClientHello` já manda key share.
- **Removeu troca de chave via RSA** — só sobrou (EC)DHE ⇒ **forward
  secrecy obrigatória**, não opcional.
- **Certificate vai cifrado** (diferença crítica para forense/captura).

**Forward secrecy:** chaves de sessão são efêmeras, descartadas após a
sessão. Comprometer a chave privada do servidor no futuro não permite
decifrar sessões antigas capturadas — a chave privada só autentica, não
decifra a conversa.

**Nomenclatura da cipher suite — TLS 1.2 vs 1.3:**
Em TLS 1.2 o nome carrega tudo (`ECDHE-RSA-AES256-GCM-SHA384`: troca de
chave, autenticação, cifra, hash) — dá pra ler PFS direto do nome. Em
TLS 1.3 o nome (`TLS_AES_256_GCM_SHA384`) só especifica cifra AEAD + hash;
a troca de chave não aparece porque **só existe uma opção** ((EC)DHE) — PFS
é garantida por eliminação de protocolo, não por leitura do nome. A
evidência direta do grupo de troca de chave efêmera aparece na linha
`Negotiated TLS1.3 group: ...` do `openssl s_client` (ex.: grupos híbridos
pós-quânticos como `X25519MLKEM768`, já em produção em provedores como
Cloudflare).

### As três garantias do TLS — mecanismo, não resumo
- **Confidencialidade** — chave simétrica de sessão cifra os dados.
- **Integridade** — AEAD/MAC verifica cada bloco; adulteração quebra a verificação.
- **Autenticação** — certificado validado pela cadeia de CAs. Sem isso, um
  atacante em MITM apresentaria a própria chave — cifrar sem autenticar
  não protege contra MITM.

## 4. O que HTTPS esconde — e o que revela

Cifra o **conteúdo**, não os metadados:
- **IP de destino** — sempre visível (camada abaixo do TLS).
- **SNI** — hostname em claro no `ClientHello`, em TLS 1.2 e 1.3 (única
  exceção: ECH — Encrypted Client Hello, adoção recente).
- **Timing, volume, frequência** dos pacotes.

Consequência para Blue Team: detecção de C2 sobre HTTPS por SNI suspeito,
certificado autoassinado/expirado/hostname que não bate (IOC), e
**fingerprint do handshake (JA3/JA3S)** — hash das escolhas do
ClientHello/ServerHello, característico de muitos malwares mesmo com
payload cifrado.

### HTTP/3 (callback Dia 6 — registro HTTPS/Type65)
HTTP/3 roda sobre **QUIC**, que roda sobre **UDP**, com TLS 1.3 embutido no
próprio transporte (não empilhado sobre TCP). Registro DNS Type65 é o
servidor anunciando suporte a HTTP/3.

## Exercícios do dia — pontos-chave da correção

1. **Anatomia HTTP crua** (`curl -v http://neverssl.com`) — leitura correta
   de request/status line e do papel do `Host`. Reforço: `>` = enviado,
   `<` = recebido no `curl -v`; `Server` no response é information
   disclosure; `Upgrade: h2,h2c` é negociação de versão HTTP nos headers.

2. **Status/métodos reais** (`curl -I`, `curl -IL`, 404 forçado) —
   `HEAD` confirmado sem corpo; redirect 301→200 identificado; ALPN
   (negociação de HTTP/1.1 vs HTTP/2 dentro do TLS) explica por que a
   versão HTTP muda entre a resposta em claro (porta 80, HTTP/1.1) e a
   resposta HTTPS (HTTP/2). Nota de processo: comando colado com prefixo
   inválido (`404: curl ...`) gerou erro de shell — reforça a regra de
   colar output real, nunca a resposta esperada.

3. **TLS handshake** (`openssl s_client`) — TLS 1.3, cipher
   `TLS_AES_256_GCM_SHA384`, cadeia de 4 certificados até
   `SSL.com TLS ECC Root CA 2022`, `Verification: OK`. PFS explicada por
   eliminação de protocolo (TLS 1.3) e confirmada na prática pela linha
   `Negotiated TLS1.3 group: X25519MLKEM768` (grupo híbrido clássico +
   pós-quântico).

4. **SNI em claro** (`tcpdump` + `curl https://example.com`) — hostname
   `example.com` confirmado em claro no dump bruto via `grep`. Ponto de
   rigor: `grep` prova presença da string, não isoladamente que veio do
   `ClientHello` (em TLS 1.2 o `Certificate` também vai em claro). Aqui a
   inferência fecha porque o Exercício 3 já provou TLS 1.3 para este
   domínio — em TLS 1.3 o `Certificate` é cifrado, então só resta o SNI
   como fonte possível.


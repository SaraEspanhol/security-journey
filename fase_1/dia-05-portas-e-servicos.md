# Dia 05 — Portas e Serviços

**Fase 1 — Fundamentos de Redes**
Conteúdo do plano: Dias 7-8 (adiantado porque TCP vs UDP fechou no Dia 4).

---

## Objetivos do dia

- Fechar o erro do diagnóstico: **porta NÃO é "entrada física"** — é um número que identifica o serviço.
- `socket = IP:porta`; a **quádrupla** que identifica uma conexão.
- Well-known ports e a leitura de segurança de cada uma.
- Portas efêmeras (cliente) vs. porta fixa (servidor).

---

## 1. O que é uma porta

Uma porta é um **número de 16 bits** (0–65535) que mora no cabeçalho do segmento TCP
(ou datagrama UDP): campo "porta de origem" e campo "porta de destino".

- O **IP identifica a máquina**; a **porta identifica o processo** dentro da máquina.
- Nada físico. Não é tomada nem conector. É um rótulo lógico dentro do pacote.
- Analogia: IP = endereço do prédio; porta = número do departamento escrito no envelope.
  O carteiro (SO) lê o número e entrega no departamento certo.

O SO mantém uma tabela "quem escuta na porta X". Chega tráfego pra 443 → entrega ao
processo que fez bind ali (ex.: nginx). Ninguém escutando → porta fechada (RST, o `[R.]`
do Dia 4).

## 2. Socket e a quádrupla

- **Socket** = `IP:porta` → um ponto de comunicação (ex.: `192.168.0.10:443`).
- Uma **conexão** = dois sockets, identificada pela **quádrupla**:

```
(IP origem, porta origem, IP destino, porta destino)
```

Um servidor aguenta milhares de conexões na MESMA porta 443 porque o SO identifica cada
conexão pela quádrupla inteira, não pela porta do servidor sozinha. Basta **um** campo
diferir pra a quádrupla ser única.

```
Conexão A: (203.0.113.5 : 54012, 192.168.0.10 : 443)
Conexão B: (198.51.100.9 : 61233, 192.168.0.10 : 443)
Conexão C: (203.0.113.5 : 54988, 192.168.0.10 : 443)  ← mesmo cliente que A, porta origem diferente
```

Caso difícil: dois clientes na **mesma** máquina → 3 dos 4 campos idênticos, só a
**porta de origem efêmera** difere. Suficiente.

## 3. Portas efêmeras

- Cliente não escolhe porta local; o SO empresta uma da faixa dinâmica, usa e devolve.
- Well-known ports = onde o **servidor** escuta, não de onde o cliente sai.

| Faixa | Nome | Uso |
|---|---|---|
| 0 – 1023 | well-known / system | serviços padrão; em Unix exige root para bind |
| 1024 – 49151 | registered | serviços registrados |
| 49152 – 65535 | dynamic / ephemeral | portas efêmeras do cliente (faixa do macOS) |

(Linux costuma usar 32768–60999 para efêmeras.)

Verificar na máquina (macOS):
```bash
sysctl net.inet.ip.portrange.first
sysctl net.inet.ip.portrange.last
```

Segurança: porta 0–1023 só root abre → serviço numa porta baixa teve privilégio elevado
em algum momento.

## 4. Well-known ports — leitura de segurança

| Porta | Serviço | O que faz | Leitura de segurança |
|---|---|---|---|
| 20/21 | FTP | transferência (21 controle, 20 dados) | **texto puro** — credenciais sem cifra |
| 22 | SSH | shell remoto cifrado (SCP/SFTP) | jeito certo; alvo nº 1 de brute force |
| 23 | Telnet | shell remoto **texto puro** | legado; LISTEN = red flag |
| 25 | SMTP | envio de e-mail (servidor→servidor) | spam / relay aberto |
| 53 | DNS | resolução nome→IP | UDP nas queries, TCP em respostas grandes; tunneling/exfil |
| 80 | HTTP | web texto puro | sem cifra; deveria redirecionar pra 443 |
| 110/143 | POP3/IMAP | recebimento de e-mail | versões antigas em texto puro |
| 443 | HTTPS | web sobre TLS | padrão cifrado (handshake vem no Dia 11-12) |
| 3389 | RDP | desktop remoto Windows | altíssimo valor p/ atacante; nunca exposto à internet |

Padrão: texto puro (FTP/Telnet/HTTP/POP3-IMAP antigos) têm primos cifrados
(SFTP/SSH/HTTPS/IMAPS). Versão texto-claro escutando = achado.

## 5. Ver portas — macOS vs Linux

macOS (armadilha: `ss` NÃO existe no macOS):
```bash
sudo lsof -iTCP -sTCP:LISTEN -n -P   # -P mostra 443, não "https"
netstat -an
```

Linux (o que se usa em SOC — guardar equivalência):
```bash
ss -tlnp     # TCP listening numérico com processo
ss -tulnp    # inclui UDP
```

`/etc/services` = dicionário nome↔número. **Descritivo, não causal**: quem abre a porta
é o processo fazendo bind. Apagar linha não fecha porta; adicionar linha não cria serviço.
Serviço pode rodar em porta que não bate com o arquivo (técnica de evasão: shell na 443
disfarçado de HTTPS). Por isso: olhar número + processo, nunca só o nome que a ferramenta mostra.

---

## Exercícios (resumo do que foi feito)

1. **`lsof` LISTEN no Mac** — mysqld em `*:3306` (todas interfaces) vs mongod em
   `127.0.0.1:27017` (só loopback). Diferença de exposição = a informação de segurança da linha.
2. **`/etc/services`** — descritivo, não causal. Bônus: encadear comandos (`;` `&&` `||`);
   `-w` casa `netconf-ssh`/`ssh-mgmt` por causa do hífen (falso positivo em log).
3. **Par de sockets (3 terminais)** — netstat mostrou os dois lados da mesma conexão +
   o socket LISTEN que continua aberto (a "recepção" sempre pronta pro próximo cliente).
   Porta efêmera lida da saída: 64476 (faixa 49152–65535).
4. **Erro do diagnóstico fechado** — porta não é física + por que 1 servidor aguenta N
   conexões (quádrupla), explicado sem consulta, sobrevivendo ao caso difícil.
5. **Triagem SOC** — Telnet (23): protocolo é o problema → quase sempre achado.
   RDP (3389): contexto é o problema → depende da exposição. Pesos diferentes.

---

## Conceitos-chave para entrevista de SOC

- "Você vê a porta X em LISTEN num servidor — é problema?" → não é o número decorado,
  é o raciocínio: que serviço / onde escuta / deveria escutar aí / está cifrado / quem abriu.
- `127.0.0.1` vs `*`/`0.0.0.0` = superfície de ataque fechada vs aberta.
- Texto-claro vs cifrado como red flag.
- **Serviço desconhecido escutando = possível backdoor** → investigar antes de assumir benigno
  (`lsof -nP -i :PORTA` + `ps -p <PID> -o pid,ppid,user,args`).
- Equivalência macOS↔Linux: `lsof`/`netstat` ↔ `ss -tlnp`.

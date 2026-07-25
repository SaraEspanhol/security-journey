# Dia 04 — TCP vs UDP

**Fase 1 — Fundamentos de Redes**
**Tópicos:** three-way handshake, TCP confiável vs. UDP sem conexão, segmentos/datagramas, portas de origem/destino, flags no tcpdump, estados de porta, SYN scan e SYN flood

---

## Teoria

### A diferença central (o tradeoff)

Os dois protocolos vivem na camada de transporte (camada 4). O trabalho deles é levar os dados ao programa certo dentro da máquina (via porta). A diferença é o *estilo de entrega*:

- **TCP** é **orientado a conexão**. Estabelece a conexão antes de trafegar dados, numera cada byte, confirma o que chega e retransmite o que falta. Troca **velocidade por confiabilidade**.
- **UDP** é **sem conexão** (*connectionless*). Dispara o pacote sem handshake, sem numeração, sem confirmação, sem retransmissão. Troca **confiabilidade por velocidade**.

Não existe "melhor" — existe adequado ao caso. O cabeçalho reflete o tradeoff: TCP tem 20+ bytes de cabeçalho (todo o controle); UDP tem 8 bytes (mínimo).

### Three-way handshake

O ritual de abertura do TCP. Três pacotes:

1. **SYN** — cliente: "quero abrir conexão, meu número de sequência inicial (ISN) é X".
2. **SYN-ACK** — servidor: "confirmo teu X (ACK), e o meu ISN é Y". Duas flags acesas porque faz duas coisas ao mesmo tempo: confirma o fluxo do cliente E abre o fluxo do servidor.
3. **ACK** — cliente: "confirmo teu Y". Conexão estabelecida.

**Por que três e não dois:** a conexão TCP é bidirecional — são dois fluxos sendo abertos, um em cada sentido. Cada lado precisa anunciar seu ISN e ter esse ISN confirmado pelo outro. O pacote do meio economiza um passo juntando "confirmo o teu" com "aqui vai o meu". O que o handshake realmente sincroniza são os **números de sequência iniciais (ISN)** dos dois lados — é isso que garante ordem e entrega, e é a base de por que TCP é confiável.

### Flags do TCP no tcpdump

Aparecem entre colchetes na captura:

| Flag | Significado |
|------|-------------|
| `[S]`  | SYN |
| `[S.]` | SYN-ACK (o `.` = ACK também ligado) |
| `[.]`  | ACK puro |
| `[P.]` | PSH-ACK (empurrando dados) |
| `[F.]` | FIN-ACK (encerrando) |
| `[R]` / `[R.]` | RST / RST-ACK (corta a conexão) |

A conexão é identificada unicamente pelo par completo `IP origem:porta origem ↔ IP destino:porta destino`. Por isso várias abas do navegador falam com o mesmo servidor na porta 443 sem misturar: cada uma usa uma **porta de origem efêmera** diferente (número alto e temporário sorteado para aquela conexão).

### Segmento vs. datagrama

A unidade de dados na camada 4 tem nomes diferentes: **segmento** no TCP, **datagrama** no UDP. Mesmo nível, vocabulário distinto.

### Quando usar cada um

- **TCP** — tudo que não pode chegar corrompido ou fora de ordem: navegação web (HTTP/HTTPS), SSH, e-mail, transferência de arquivo, internet banking. Um byte perdido corrompe o todo → precisa de garantia. TLS roda sobre TCP justamente porque criptografia exige bytes íntegros e em ordem.
- **UDP** — tudo onde chegar rápido importa mais que chegar perfeito: VoIP, videochamada, streaming ao vivo, jogos, **consulta DNS**. Retransmitir um pacote de áudio já vencido não tem valor — quando chegasse, o momento já passou. A latência é o inimigo, não a perda.

**Detalhe sobre DNS:** usa UDP por padrão (2 pacotes resolvem uma consulta que caberia em datagramas pequenos; TCP gastaria 7+ com o handshake). Mas *cai para TCP* quando a resposta é grande demais para o datagrama ou em transferência de zona. DNS sobre TCP em volume anômalo é sinal de alerta em SOC (tentativa de transferência de zona não autorizada ou exfiltração via DNS).

---

## Exercícios — resumo dos resultados

| Exercício | Tema | Resultado |
|---|---|---|
| 1 | Capturar three-way handshake real (`tcpdump`) | Fechado após aprofundamento — captura correta (inclusive em IPv6, sem hesitar); refinado o papel duplo do `[S.]` e o conceito de ISN |
| 2 | Conexão vs. sem-conexão (`nc` TCP vs UDP) | Fechado após correção de método — resposta inicial deduzida, não observada; refeito rodando `nc` + `tcpdump` |
| 3 | Ler trecho de captura (SYN → `[R.]` na porta 22) | Fechado, sem ressalvas — leitura por contraste com SYN-ACK esperado; identificou port scan / reconnaissance |
| 4 | Escolher protocolo e justificar pelo tradeoff | Fechado, sem ressalvas — justificativas ligadas a ordem/perda/velocidade nos 4 cenários |
| 5 (opcional) | Por que SYN flood derruba servidor | Fechado — identificou a premissa correta: servidor reserva estado antes de confirmar a conexão |

---

## Descobertas práticas

### O "succeeded!" do UDP — sucesso local vs. remoto

`nc -u -v 127.0.0.1 9999` imprime `Connection to ... succeeded!` **mesmo sem ninguém escutando** na porta. A palavra "succeeded" é enganosa: em UDP não há handshake, então o `nc` não confirma nada com o outro lado — ele só reporta que o próprio SO configurou o socket e está pronto para *tentar* enviar. É **sucesso local** ("consigo enviar"), não **sucesso remoto** ("há um serviço ouvindo"). A captura no loopback provou isso: quatro datagramas saindo (`127.0.0.1.52737 > 127.0.0.1.9999`), zero voltando. Nenhuma resposta — nem SYN-ACK (não existe em UDP), nem ICMP (o loopback do macOS não gera). Silêncio absoluto, e mesmo assim "succeeded!" na tela.

Contraste com TCP: porta fechada devolve `RST` na hora → `nc` diz `Connection refused`. O sucesso do TCP é remoto por construção; o "sucesso" do UDP é local e vazio como prova.

### Três estados de porta

- **Aberta** — responde com SYN-ACK (`[S.]`).
- **Fechada** — responde com RST (`[R.]`).
- **Filtrada** — **nada volta**. Firewall no caminho fez *drop* silencioso. É o mesmo silêncio do UDP: ausência de resposta não prova aberta nem fechada — só prova que não se sabe.

Regra que atravessa o dia: **ausência de resposta não é uma resposta.** Num scan UDP, não receber nada pode ser porta fechada com ICMP bloqueado, porta aberta e calada, ou pacote perdido — três realidades, o mesmo silêncio. Por isso scan UDP é lento e ambíguo, e scan TCP é rápido e limpo (o RST vem por construção do protocolo).

---
## Conexão com o mercado (SOC N1)

Ler um three-way handshake numa captura é habilidade básica e diária de SOC. Três aplicações diretas vistas hoje:

- **Port scan / SYN scan (reconnaissance)** — muitos SYN de um mesmo IP para portas diferentes em segundos é a assinatura clássica. O *SYN scan* (half-open, `nmap -sS`) corta antes do terceiro passo para ser silencioso — usa o fato de que a conexão só "existe" após o ACK final para não gerar log.
- **SYN flood (DoS)** — mesmo mecanismo (SYN sem ACK final), fim oposto: o atacante nunca completa o handshake, forçando o servidor a manter conexões meio-abertas na fila (**backlog**) até esgotar recurso. Defesa clássica: **SYN cookies** — o servidor não reserva estado até a conexão se provar real, desarmando a premissa do ataque.
- **TCP vs UDP num alerta** — scans e tráfego UDP se interpretam diferente de TCP (sem SYN/RST para observar). Volta na Fase 5 (Ataques) e na Fase 7 (Wazuh).

Triagem que um N1 faz de imediato: distinguir handshake normal de handshake quebrado (RST inesperado, SYN sem conclusão, enxurrada de SYN) — só possível para quem entende que o handshake tem exatamente três passos e que qualquer um deles pode faltar de propósito.

---

**Próximo dia:** Dia 5 — Portas e serviços (porta = número que identifica o serviço; socket = IP:porta; well-known ports: 20/21, 22, 23, 25, 53, 80, 110/143, 443, 3389).
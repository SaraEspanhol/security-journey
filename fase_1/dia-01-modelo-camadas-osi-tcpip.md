# Dia 01 — Modelo em Camadas: OSI e TCP/IP

**Fase 1 — Fundamentos de Redes**
**Data:** 21 JUL
**Tópicos:** por que redes usam camadas, modelo OSI (7 camadas), modelo TCP/IP (4 camadas), encapsulamento/desencapsulamento

---

## Teoria

### Por que camadas existem

Separação de responsabilidades com contrato fixo: cada camada resolve um problema específico, oferece um serviço para a camada de cima e consome o serviço da camada de baixo, sem precisar saber *como* a camada de baixo funciona — só *o que* ela entrega. Isso permite trocar a implementação de uma camada (ex.: Wi-Fi por cabo) sem afetar as demais.

Relevância em segurança: ataques e defesas moram em camadas específicas. Um firewall de camada 4 não enxerga um SQL Injection porque o SQL está no payload da camada 7, que ele nem abre. Localizar a camada de um problema é o primeiro passo de triagem de qualquer incidente.

### Modelo OSI (7 camadas) — modelo de referência / vocabulário da indústria

| # | Camada | O que resolve | PDU | Exemplos |
|---|--------|---------------|-----|----------|
| 1 | Física | transmitir bits pelo meio | bit | cabo, fibra, rádio |
| 2 | Enlace | entrega no mesmo segmento local | quadro (frame) | Ethernet, Wi-Fi, MAC, switch, ARP |
| 3 | Rede | endereçar e rotear entre redes | pacote | IP, ICMP, roteador |
| 4 | Transporte | entrega fim a fim, identificar programa | segmento/datagrama | TCP, UDP, portas |
| 5 | Sessão | abrir/manter/encerrar diálogo | dados | controle de sessão |
| 6 | Apresentação | tradução, codificação, cifragem | dados | UTF-8, JPEG |
| 7 | Aplicação | significado dos dados p/ usuário | dados | HTTP, DNS, SSH |

Pontos-chave:
- MAC = endereço plano, de fábrica, só tem significado no segmento local.
- IP = endereço hierárquico (rede + host), permite roteamento em escala global sem que cada roteador conheça cada dispositivo do mundo.
- Camada 7 não é o programa (o Chrome não é camada 7; o HTTP que ele fala é).
- TLS é tradicionalmente associado à camada 6, mas na prática opera entre a 4 e a 7 — sintoma de que o OSI é vocabulário, não implementação literal.

### Modelo TCP/IP (4 camadas) — o que roda de fato

Aplicação (7+6+5 do OSI) → Transporte (4) → Internet (3) → Acesso à rede (2+1).

OSI = régua de vocabulário (usado em prova/Security+). TCP/IP = implementação real (o que o Wireshark mostra).

### Encapsulamento

Ao descer a pilha, cada camada recebe o dado de cima como payload opaco e acrescenta seu próprio cabeçalho na frente (camada 2 também acrescenta um trailer, o FCS).

```
Aplicação  [ dados ]
Transporte [ TCP hdr ][ dados ]                     → segmento
Rede       [ IP hdr ][ TCP hdr ][ dados ]            → pacote
Enlace     [ Eth hdr ][ IP hdr ][ TCP hdr ][ dados ][FCS] → quadro
Física     bits
```

Cabeçalho vai na frente porque cada dispositivo intermediário precisa ler a informação de controle **antes** de decidir o que fazer com o resto — processamento é sequencial. Um switch decide a porta de saída lendo o cabeçalho Ethernet sem esperar o quadro inteiro chegar.

Desencapsulamento é o processo inverso na recepção: cada camada remove seu cabeçalho e entrega o resto para a camada de cima. A camada 7 do destino recebe exatamente os bytes que a camada 7 da origem enviou.

### ICMP e a ausência das camadas 4-7

Ping usa ICMP, que é o protocolo de controle da própria camada 3 (reporta problemas de rede, não carrega dados de aplicação). Não há camada 4 porque ICMP não precisa de porta (não identifica programa/aplicação) nem de entrega confiável com retransmissão (a perda do pacote É a informação útil — mede conectividade). Por não ter porta, ICMP não pode ser filtrado por porta em firewall; é tratado como regra à parte.

### MAC vs IP ao longo de um trajeto (hop-by-hop forwarding)

- IP de destino: permanece igual do início ao fim (resolve o problema global — "para qual máquina na internet estou indo").
- MAC de destino: muda a cada salto (resolve o problema local — "para qual vizinho entrego este quadro agora"). Cada roteador desencapsula o quadro, decide o próximo salto pelo IP, e reencapsula com novo MAC origem (própria interface) e novo MAC destino (próximo salto). O protocolo que resolve IP→MAC em cada enlace é o ARP (formalizado nos Dias 26-27).

---

## Exercícios — resumo dos resultados

| Exercício | Tema | Resultado |
|---|---|---|
| 1 | Identidades da máquina (MAC/IP/gateway) | Fechado |
| 2 | Encapsulamento observado no Wireshark | Fechado |
| 3 | Triagem por camada (4 cenários) | Fechado, 4/4 sem ressalvas |
| 4 | Descrever encapsulamento de memória | Fechado, sem ressalvas |
| 5 (opcional) | MAC vs IP ao longo do trajeto | Fechado, sem ressalvas |


## Conexão com o mercado (SOC N1)

Pergunta clássica de entrevista de SOC N1: "o usuário reporta que não consegue acessar um sistema — como você investiga?" Resposta estruturada por camada (física → enlace → IP → DNS → aplicação) demonstra processo de triagem, diferente de resposta sem estrutura ("eu vejo os logs").

---

## Captura Wireshark — Exercício 2

Pacote ICMP (ping para 8.8.8.8), filtro `icmp`.

Estrutura das camadas no painel de detalhes:
- Frame (metadados da captura, não é camada OSI) — 98 bytes total
- Ethernet II — camada 2 (Enlace)
- Internet Protocol Version 4 — camada 3 (Rede) — cabeçalho de 20 bytes
- Internet Control Message Protocol — camada 3 (Rede), protocolo de controle
- Data (payload ICMP, bytes de enchimento sem significado de aplicação)

Camadas 4 a 7 ausentes — ICMP não precisa de porta nem de entrega confiável.

---

**Próximo dia:** Dia 2 — cliente/servidor, LAN/WAN, percurso ponta a ponta completo.
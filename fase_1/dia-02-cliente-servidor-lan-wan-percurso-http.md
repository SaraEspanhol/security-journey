# Dia 02 — Cliente/Servidor, LAN/WAN e Percurso Ponta a Ponta

**Fase 1 — Fundamentos de Redes**
**Tópicos:** cliente/servidor como papel, LAN vs WAN, percurso completo DNS → TCP → TLS → HTTP

---

## Teoria

### Cliente e servidor são papéis, não categorias de máquina

Servidor é quem **espera** conexão e oferece um serviço numa porta conhecida (estado `LISTEN`). Cliente é quem **inicia** a conexão e consome o serviço (`ESTABLISHED` saindo para porta remota). Qualquer máquina pode assumir os dois papéis em contextos diferentes — ex.: um servidor de banco é servidor para o usuário final, mas cliente quando chama uma API externa de verificação de fraude.

Relevância para triagem de incidente: identificar se uma máquina comprometida está *recebendo* comando (agindo como servidor) ou *chamando* um servidor de controle remoto (agindo como cliente) é literalmente essa distinção aplicada.

### LAN, WAN e a internet como "rede de redes"

- **LAN**: rede confinada fisicamente pequena (casa, escritório). Dispositivos se enxergam direto na camada 2 (MAC), sem roteador no meio, por compartilharem o mesmo segmento.
- **WAN**: conecta LANs geograficamente distantes. A internet é uma WAN, mas é mais preciso dizer que é uma **rede de redes** — milhares de LANs interligadas por roteadores que concordam em encaminhar tráfego entre si.
- Dentro da LAN: resolução por MAC (camada 2). Saindo para a WAN: roteamento IP (camada 3), hop-by-hop, via gateway.
- MAN (Metropolitan Area Network): escala intermediária (cidade). Pouco citada no dia a dia de SOC júnior, mas aparece em prova.

### O percurso ponta a ponta completo (URL HTTPS → página na tela)

1. **DNS** (camada 7, mas de infraestrutura): navegador resolve nome → IP consultando um servidor DNS. Etapa que acontece **antes** de qualquer conexão com o site em si.
2. **TCP three-way handshake** (camada 4): SYN (cliente) → SYN-ACK (servidor) → ACK (cliente). Só depois da conexão estabelecida é que dados reais podem trafegar. Base do ataque SYN flood (Fase 5).
3. **TLS handshake** (se HTTPS): Client Hello (cliente propõe algoritmos) → Server Hello + certificado (servidor escolhe algoritmo e prova identidade) → negociação de chave de sessão. Depois disso o canal está cifrado (cadeado no navegador). O **SNI** (Server Name Indication), campo dentro do Client Hello, viaja em texto claro e revela para qual domínio o cliente está se conectando — mesmo sem decifrar o resto do tráfego.
4. **Requisição/resposta HTTP** (camada 7): `GET /` + cabeçalhos; servidor responde com status code + conteúdo.
5. **Encerramento** (FIN/ACK, não detalhado hoje).

Cada etapa depende da anterior ter terminado — não há como mandar HTTP antes do TCP estabelecido, nem TLS antes do TCP.

### QUIC/HTTP-3 (nota lateral, fora do escopo de hoje)

Protocolo mais recente que roda sobre **UDP**, não TCP, usado por grandes players (Google, Apple) para evitar overhead do three-way handshake clássico. Apareceu na captura do dia como ruído de fundo — sinal de que tráfego de sistema/apps modernos frequentemente usa isso em vez de TCP+TLS clássico.

---

## Exercícios — resumo dos resultados

| Exercício | Tema | Resultado |
|---|---|---|
| 1 | Cliente ou servidor em 4 cenários | Fechado, 4/4 sem ressalvas |
| 2 | LAN na prática (`arp -a`) | Fechado, sem ressalvas |
| 3 | Captura do percurso completo (curl -v + Wireshark) | **Não fechado** — item 1 correto; item 2 (isolar handshake TCP+TLS de `example.com` no Wireshark) não conseguiu ser isolado apesar de várias tentativas; item 3 com explicação teórica correta mas sem evidência de captura válida. Pulado por decisão conjunta. |
| 4 | Percurso completo sem consulta | Fechado, sem ressalvas |
| 5 (opcional) | Ordem TCP → TLS obrigatória | Pulado (opcional) |

## Pendência registrada — Fragilidade persistente da Fase 1

**Habilidade de isolar um fluxo (`tcp.stream`) específico numa captura Wireshark com tráfego de fundo real.** Não é falha conceitual — a teoria do percurso (DNS→TCP→TLS→HTTP) foi corretamente descrita e defendida (Exercícios 1 e 4 sem ressalvas). É falta de prática mecânica com filtros de captura em ambiente ruidoso (múltiplas conexões simultâneas, QUIC, tráfego de sistema/Apple competindo).

Relevância futura: essa é a mesma habilidade central de "isolar o sinal relevante dentro do ruído" que será exigida com força na Fase 3 (análise de logs) e na Fase 7 (SIEM/Wazuh, triagem de alertas). Retomar com exercício dedicado, em ambiente mais controlado, antes ou durante a Fase 3.

## Conexão com o mercado (SOC N1)

O percurso DNS→TCP→TLS→HTTP é pergunta clássica de entrevista técnica júnior. Cliente/servidor como papel (não hardware) é o vocabulário usado ao descrever conexões suspeitas em triagem de incidente (ex.: máquina comprometida como cliente de C2 vs. servindo backdoor). SNI como identificador visível em tráfego cifrado é técnica real usada em proxies corporativos e alguns firewalls.

---

**Próximo dia:** Dia 3 — Endereçamento IP (IPv4, IP público vs. privado, máscara/sub-rede básico, NAT, gateway, DHCP).
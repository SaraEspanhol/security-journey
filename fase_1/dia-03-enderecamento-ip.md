# Dia 03 — Endereçamento IP

**Fase 1 — Fundamentos de Redes**
**Tópicos:** IPv4, IP público vs. privado, máscara de sub-rede/CIDR, NAT, gateway, DHCP

---

## Teoria

### IPv4 — estrutura

Endereço de 32 bits, escrito em 4 octetos decimais (0-255) separados por ponto. `2^32` ≈ 4,3 bilhões de combinações — espaço já esgotado, daí a necessidade de IP privado + NAT como remendo estrutural. IPv6 (128 bits) é a solução definitiva, já em uso real (apareceu nas capturas dos dias anteriores via dual-stack).

### IP público vs. privado — diferença estrutural, não de segurança

Blocos reservados por RFC 1918, roteadores da internet descartam pacotes destinados a esses blocos vindos de fora:

```
10.0.0.0/8        (redes grandes)
172.16.0.0/12     (redes médias)
192.168.0.0/16    (redes domésticas — bloco mais comum)
```

IP privado só precisa ser único **dentro** da própria LAN — pode ser reutilizado em milhões de redes diferentes ao mesmo tempo, porque cada uma é isolada da outra pelo próprio roteador/NAT. IP público precisa ser único no mundo inteiro.

### NAT (Network Address Translation)

Processo do roteador que traduz IP privado interno ↔ IP público externo, mantendo uma tabela de conexões (IP interno + porta ↔ conexão específica) para rotear as respostas de volta corretamente. Do ponto de vista da internet, só o roteador existe — todos os dispositivos internos são invisíveis individualmente.

Efeito colateral de segurança (não é o propósito do NAT, mas acontece): como NAT só cria entrada na tabela para conexões **iniciadas de dentro para fora**, tráfego iniciado de fora para dentro é descartado por padrão — não há entrada correspondente. É por isso que hospedar um serviço próprio em casa (ex.: servidor web pessoal) não é acessível de fora sem **port forwarding**: uma regra estática manual que cria uma entrada permanente na tabela NAT, direcionando `IP_público:porta` para `IP_privado:porta` mesmo sem conexão de saída prévia.

### Máscara de sub-rede e CIDR

A máscara (32 bits, bits `1` = parte de rede, bits `0` = parte de host) define onde termina a parte de rede e começa a parte de host dentro do IP. `255.255.255.0` = `/24` (24 bits em 1, contando da esquerda).

Teste prático: aplicar a mesma máscara a dois IPs e comparar o resultado (endereço de rede). Se iguais → mesma rede, vizinho local (resolução via ARP/camada 2). Se diferentes → redes diferentes, precisa de gateway (roteamento via camada 3). É esse teste que a máquina faz internamente para decidir se manda o pacote direto ou via gateway.

### DHCP (Dynamic Host Configuration Protocol)

Serviço (normalmente no próprio roteador doméstico) que distribui automaticamente IP, máscara, gateway e DNS para dispositivos que entram na rede. Processo **DORA**:

1. **Discover** — cliente pergunta em broadcast (ainda não tem IP).
2. **Offer** — servidor oferece um IP disponível.
3. **Request** — cliente confirma que aceita.
4. **Acknowledge (ACK)** — servidor confirma a concessão, com lease time (tempo de validade).

Consequência de segurança: DHCP responde a qualquer dispositivo sem autenticação por padrão — base do ataque **rogue DHCP / DHCP spoofing** (servidor DHCP falso distribui gateway malicioso, redirecionando tráfego da vítima). Aprofundado na Fase 5.

### Quadro completo — rede doméstica típica

Roteador acumula três papéis num único dispositivo: gateway (`192.168.1.1`), servidor DHCP, servidor DNS local. Padrão universal em roteadores consumer; em redes corporativas geralmente esses serviços são distribuídos.

---

## Exercícios — resumo dos resultados

| Exercício | Tema | Resultado |
|---|---|---|
| 1 | CIDR, endereço de rede, gateway na mesma rede | Fechado, sem ressalvas |
| 2 | IP público real (NAT) via `curl ifconfig.me` | Fechado, sem ressalvas |
| 3 | DHCP na prática (`ipconfig getpacket`) | Fechado, sem ressalvas — identificação completa de todos os parâmetros e do papel triplo do roteador |
| 4 | Por que port forwarding é necessário | Fechado, sem ressalvas |
| 5 (opcional) | Reuso de IP privado entre redes diferentes | Fechado, sem ressalvas |

---

**Próximo dia:** Dia 4 — continuação de Endereçamento IP (consolidação prática) ou Dias 5-6, TCP vs UDP (conforme plano programático, a confirmar com o professor no próximo prompt).
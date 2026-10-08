# Respostas do questionário

## Etapa 1 - Matriz

### Q1. A conversa DHCP completa

Na LAN-ENG, a troca DHCP observada foi:

| Mensagem | IP de origem | IP de destino | MAC de destino |
|---|---|---|---|
| DISCOVER | 0.0.0.0 | 255.255.255.255 | FFFF.FFFF.FFFF |
| OFFER | 10.1.1.65 | 255.255.255.255 | FFFF.FFFF.FFFF |
| REQUEST | 0.0.0.0 | 255.255.255.255 | FFFF.FFFF.FFFF |
| ACK | 10.1.1.65 | 255.255.255.255 | FFFF.FFFF.FFFF |

O DISCOVER parte de `0.0.0.0` porque o PC ainda não tem um endereço IP. O REQUEST continua com essa origem porque o endereço oferecido ainda não foi confirmado. As duas mensagens usam broadcast porque o PC ainda não conhece o servidor DHCP.

Evidências: `evidencias/prints/q1_dhcp_dora.png` e `evidencias/prints/q1_pdu_discover.png`. As capturas complementares da oferta, solicitação e confirmação também estão na mesma pasta.

### Q2. O relay em ação

Antes do MTZ-R2, o DHCPDISCOVER tem origem `0.0.0.0` e destino `255.255.255.255`. Depois do relay, a origem passa para `10.1.1.65` e o destino passa para `10.1.1.146`, que é o IP do SRV-DC.

O campo GIADDR, mostrado no Packet Tracer como Relay Agent Address, recebe `10.1.1.65`. Esse IP informa ao servidor que o pedido veio da rede `10.1.1.64/27`. Assim, o servidor usa o pool da LAN-ENG.

Evidências: `evidencias/prints/q2_giaddr.png` e `evidencias/prints/q2_relay_ips.png`.

### Q3. E sem o relay?

Sem o `ip helper-address`, o DHCPDISCOVER saiu do PC, passou pelo SW-ENG e parou no MTZ-R2. O roteador não repassa broadcasts entre sub-redes. O relay resolve isso ao encaminhar o pedido diretamente para o servidor DHCP.

Depois do teste, o relay foi configurado novamente:

```text
MTZ-R2#show ip interface gigabitEthernet 0/2
GigabitEthernet0/2 is up, line protocol is up (connected)
  Internet address is 10.1.1.65/27
  Helper address is 10.1.1.146
```

Evidências: `evidencias/prints/q3_discover_descartado.png` e `evidencias/prints/q3_sem_relay_eventos.png`.

### Q4. DR e BDR onde não se esperava

O MTZ-R1 apresentou dois vizinhos em estado FULL:

```text
MTZ-R1#show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.2           1   FULL/DR         00:00:36    10.1.1.154      GigabitEthernet0/0
1.1.1.3           1   FULL/DR         00:00:36    10.1.1.158      GigabitEthernet0/1
```

Significado das colunas:

- `Neighbor ID`: Router ID do vizinho;
- `Pri`: prioridade usada na eleição de DR e BDR;
- `State`: estado da vizinhança e função do vizinho;
- `Dead Time`: tempo restante para considerar o vizinho indisponível;
- `Address`: IP da interface do vizinho;
- `Interface`: interface local usada para chegar ao vizinho.

O enlace GigabitEthernet usa o tipo de rede broadcast no OSPF. Por isso existe eleição de DR e BDR mesmo com apenas dois roteadores.

O OSPF foi configurado na ordem MTZ-R1, MTZ-R2 e MTZ-R3. O MTZ-R2 ficou como DR no enlace R1-R2. O MTZ-R3 ficou como DR nos enlaces R1-R3 e R2-R3. Todas as prioridades são 1 e, nesta montagem, o maior Router ID venceu as três eleições.

O DR não é substituído só porque apareceu um roteador com Router ID maior. Para forçar outra eleição, seria necessário reiniciar o processo OSPF nos roteadores do enlace com `clear ip ospf process`.

Evidências: saída acima, `evidencias/configs/etapa1_MTZ-R1.txt`, `evidencias/configs/etapa1_MTZ-R2.txt` e `evidencias/configs/etapa1_MTZ-R3.txt`.

### Q5. Anatomia de [110/2]

Na tabela do MTZ-R2, a rota para a LAN-SRV aparece assim:

```text
MTZ-R2#show ip route
O       10.1.1.144/29 [110/2] via 10.1.1.153, GigabitEthernet0/0
```

O `110` é a distância administrativa do OSPF. O `2` é o custo da rota. O caminho usa a saída do MTZ-R2 para o MTZ-R1, com custo 1, e a saída do MTZ-R1 para a LAN-SRV, também com custo 1. O total é 2.

```text
MTZ-R1#show ip ospf interface gigabitEthernet 0/0
  Internet address is 10.1.1.153/30, Area 0
  Network Type BROADCAST, Cost: 1

MTZ-R1#show ip ospf interface gigabitEthernet 0/2
  Internet address is 10.1.1.145/29, Area 0
  Network Type BROADCAST, Cost: 1
```

Com a referência padrão de 100 Mb/s, FastEthernet e GigabitEthernet ficam com custo 1. Isso significa que o OSPF não diferencia enlaces acima de 100 Mb/s nessa configuração. Para diferenciar as velocidades, a referência de banda deve ser aumentada em todos os roteadores.

Evidências: blocos de comandos acima e `evidencias/configs/etapa1_MTZ-R1.txt`.

### Q6. O batimento cardíaco do OSPF

O intervalo Hello é de 10 segundos e o Dead é de 40 segundos:

```text
MTZ-R1#show ip ospf interface gigabitEthernet 0/0
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
```

Na simulação, um Hello apareceu em `11.217 s` e o próximo em `21.217 s`. A diferença foi de 10 segundos.

Se o roteador não receber Hello do vizinho durante 40 segundos, a vizinhança é removida e as rotas são recalculadas.

Evidência: `evidencias/prints/q6_hellos.png`.

## Etapa 2 - Filial legada

### Q7. Os quatro relógios do RIP

O `show ip protocols` de FIL-R1 mostra os quatro timers:

```text
Sending updates every 30 seconds, next due in 19 seconds
Invalid after 180 seconds, hold down 180, flushed after 240
```

| Timer | Valor | Papel |
|---|---:|---|
| Update | 30 s | Intervalo entre os envios periódicos da tabela de rotas aos vizinhos. |
| Invalid | 180 s | Se uma rota não for anunciada nesse tempo, ela é marcada como inválida (métrica 16). |
| Holddown | 180 s | Depois que uma rota fica inválida, o roteador ignora anúncios piores sobre ela nesse período, para evitar loops. |
| Flush | 240 s | Se a rota continuar sem anúncio, ela é removida da tabela. |

A linha `next due in 19 seconds` mostra quanto tempo falta para o próximo update periódico de FIL-R1.

Evidência: `evidencias/etapa2.pdf`, seção 1.2 (`show ip protocols` em FIL-R1).

### Q8. Por dentro de um Response

O RIP Response periódico de FIL-R1 tem destino `224.0.0.9`, com origem `10.1.1.177` e porta UDP 520. No cabeçalho RIP, `CMD:0x02` indica Response e `VER:0x02` indica RIPv2.

O RIPv2 usa o multicast `224.0.0.9` porque só os roteadores RIPv2 processam esse grupo. O RIPv1 usava broadcast `255.255.255.255`, que obriga todos os equipamentos da rede a receber e processar o pacote.

Cada entrada de rota do Response tem estes campos:

| Campo | Valor no print |
|---|---|
| Address Family | 2 (IP) |
| Route Tag | 0 |
| Network Address | 10.1.1.96 |
| Subnet Mask | 255.255.255.224 |
| Next Hop | 10.1.1.177 |
| Metric | 1 |

O campo Subnet Mask é a razão de o RIPv2 suportar VLSM. Cada rota leva a sua própria máscara. Por isso, `/27`, `/28` e `/30` convivem dentro da rede `10.0.0.0`. O RIPv1 não envia máscara, então o receptor teria de deduzir a máscara pela classe do endereço.

O Response tem só uma entrada, a LAN-FIL1. Pelo split horizon, FIL-R1 não anuncia pela serial a rede `10.1.1.176/30` dessa própria interface nem a `10.1.1.128/28`, que aprendeu de FIL-R2.

Evidência: `evidencias/prints/q8_rip_response.png`.

### Q9. Métrica em saltos

```text
FIL-R2#show ip route rip
     10.0.0.0/8 is variably subnetted, 5 subnets, 4 masks
R       10.1.1.96/27 [120/1] via 10.1.1.177, 00:00:13, Serial0/3/0
```

O `120` é a distância administrativa do RIP. O `1` é a métrica: existe um salto até a LAN-FIL1, que é FIL-R1. A rota foi aprendida pela `Serial0/3/0`, com próximo salto `10.1.1.177`.

O RIP é um protocolo de vetor de distância simples. Ele mede o caminho pela quantidade de roteadores e não considera a velocidade dos enlaces.

A métrica 16 significa rede inalcançável. Na prática, o RIP só alcança destinos a até 15 saltos. Esse limite também encerra a contagem ao infinito durante falhas, mas impede o uso do RIP em redes com cadeias maiores de roteadores.

### Q10. Tagarelice comparada

Com a rede estável, cada protocolo continua enviando mensagens periódicas:

| | RIP (filial) | OSPF (matriz) |
|---|---|---|
| Mensagem | Response com a tabela de rotas | Hello |
| Destino | 224.0.0.9 | 224.0.0.5 |
| Intervalo | 30 s | 10 s |
| Tamanho do pacote IP | 52 bytes | 20 + 48 = 68 bytes |
| Mensagens por minuto | 2 | 6 |
| Bytes por minuto, por roteador e enlace | 2 × 52 = 104 | 6 × 68 = 408 |

No print do RIP, FIL-R1 enviou Responses em `3.850 s` e `29.809 s`. Na sequência do print da Q8, os envios de FIL-R1 aparecem em `4.010`, `31.262`, `61.113`, `87.809`, `117.680` e `144.509 s`. Os intervalos ficam entre 26 e 30 s porque o IOS aplica uma pequena variação aleatória ao timer de 30 s. Isso evita que todos os roteadores enviem updates ao mesmo tempo.

Os 52 bytes do Response são 20 de cabeçalho IP, 8 de UDP, 4 de cabeçalho RIP e 20 da única entrada de rota. Cada nova rota acrescenta 20 bytes ao Response, e ele é reenviado inteiro a cada 30 s. Assim, o gasto do RIP cresce junto com a tabela de rotas.

No print do OSPF, o Hello de MTZ-R2 tem `TYPE:1`, destino `224.0.0.5` e `PACKET LENGTH:48`. O Packet Tracer mostra `TL:20` no cabeçalho IP desse pacote, que corresponde só ao cabeçalho IP. Por isso o tamanho foi calculado como 20 bytes de IP mais os 48 bytes do OSPF, ou seja, 68 bytes. O intervalo de 10 s entre Hellos foi medido na Q6 (`11.217 s` e `21.217 s`).

Os 48 bytes do Hello são 24 de cabeçalho OSPF, 20 de campos fixos do Hello e 4 para o único vizinho do enlace. O Hello do OSPF não carrega rotas. Seu tamanho depende só da quantidade de vizinhos no enlace. Por isso, o tráfego periódico do OSPF continua igual quando a rede ganha novas sub-redes. O OSPF só envia informação de rotas quando há mudança na topologia.

Nesta medição, o OSPF gasta mais bytes por minuto do que o RIP (408 contra 104), porque a filial tem poucas rotas e o Hello é enviado com mais frequência. Esse resultado inverte quando a rede cresce. Cada rota nova acrescenta 20 bytes ao Response do RIP, e a partir de 9 rotas o RIP passa a gastar mais (2 × (32 + 20 × 9) = 424 bytes por minuto). O Hello do OSPF continua com 68 bytes. O protocolo que cresce com a tabela de rotas é o RIP.

Evidências: `evidencias/prints/q10_rip_periodico.png`, `evidencias/prints/q10_ospf_periodico.png`, `evidencias/prints/q8_rip_response.png` e `evidencias/prints/q6_hellos.png`.

## Etapa 3 - Integração

### Q11. O network classful do RIP

O comando `network 10.0.0.0` aceita só a rede classful. Por isso, em R-BORDA ele ativa o RIP em todas as interfaces com endereço 10.x.x.x: `Serial0/0/0` (10.1.1.166, para MTZ-R2), `Serial0/0/1` (10.1.1.170, para MTZ-R3) e `Serial0/1/0` (10.1.1.173, para FIL-R1).

Isso é um problema em um roteador de borda, porque todas as interfaces dele estão dentro de 10.0.0.0/8. Sem nenhum ajuste, R-BORDA enviaria updates RIP também para a matriz, que roda só OSPF.

O `passive-interface` impede que o roteador envie updates RIP pela interface. Ele não retira a rede da interface do processo: a rede continua sendo anunciada pelas outras interfaces, e o roteador continua podendo receber updates por ali.

A saída mostra que o RIP envia e recebe só pela `Serial0/1/0`, e que as duas seriais da matriz estão como passivas:

```text
R-BORDA#show ip protocols
Routing Protocol is "rip"
Sending updates every 30 seconds, next due in 10 seconds
Invalid after 180 seconds, hold down 180, flushed after 240
Outgoing update filter list for all interfaces is not set
Incoming update filter list for all interfaces is not set
Redistributing: rip, ospf 1
Default version control: send version 2, receive 2
  Interface             Send  Recv  Triggered RIP  Key-chain
  Serial0/1/0           22
Automatic network summarization is not in effect
Maximum path: 4
Routing for Networks:
	10.0.0.0
Passive Interface(s):
	Serial0/0/0
	Serial0/0/1
```

Evidência: `evidencias/etapa3.pdf`, seção 1.1.

### Q12. A métrica-semente que faltou

Com `redistribute ospf 1 metric 16`, as rotas da matriz não apareceram em FIL-R1. A tabela só tinha as redes da filial e os dois /30 entre R-BORDA e a matriz:

```text
FIL-R1#show ip route
     10.0.0.0/8 is variably subnetted, 9 subnets, 4 masks
C       10.1.1.96/27 is directly connected, GigabitEthernet0/0
L       10.1.1.97/32 is directly connected, GigabitEthernet0/0
R       10.1.1.128/28 [120/1] via 10.1.1.178, 00:00:22, Serial0/3/0
R       10.1.1.164/30 [120/1] via 10.1.1.173, 00:00:19, Serial0/3/1
R       10.1.1.168/30 [120/1] via 10.1.1.173, 00:00:19, Serial0/3/1
C       10.1.1.172/30 is directly connected, Serial0/3/1
L       10.1.1.174/32 is directly connected, Serial0/3/1
C       10.1.1.176/30 is directly connected, Serial0/3/0
L       10.1.1.177/32 is directly connected, Serial0/3/0
```

Nesse momento, o `show ip protocols` de R-BORDA já mostrava `Redistributing: rip, ospf 1`. A redistribuição estava ativa, então a causa era a métrica. No RIP, 16 significa inalcançável. As rotas da matriz foram anunciadas para FIL-R1 como inalcançáveis e descartadas.

Depois da troca para `redistribute ospf 1 metric 3`, as rotas da matriz entraram em FIL-R1 com métrica 3:

```text
FIL-R1#show ip route
R       10.1.1.0/26 [120/3] via 10.1.1.173, 00:00:15, Serial0/3/1
R       10.1.1.64/27 [120/3] via 10.1.1.173, 00:00:15, Serial0/3/1
R       10.1.1.144/29 [120/3] via 10.1.1.173, 00:00:15, Serial0/3/1
R       10.1.1.152/30 [120/3] via 10.1.1.173, 00:00:15, Serial0/3/1
R       10.1.1.156/30 [120/3] via 10.1.1.173, 00:00:15, Serial0/3/1
R       10.1.1.160/30 [120/3] via 10.1.1.173, 00:00:15, Serial0/3/1
```

Em um roteador Cisco real, omitir o parâmetro `metric` teria o mesmo efeito da métrica 16. Sem métrica definida, a semente padrão para rotas redistribuídas no RIP é infinita, porque custo OSPF não pode ser convertido em saltos. As rotas seriam anunciadas como inalcançáveis e não entrariam na tabela do vizinho.

Evidências: `evidencias/etapa3.pdf`, seções 3.1 e 3.2.

### Q13. Rotas de segunda mão no OSPF

As redes da filial aparecem em MTZ-R1 com o código `O E2`:

```text
MTZ-R1#show ip route
O E2    10.1.1.96/27 [110/20] via 10.1.1.154, 00:07:03, GigabitEthernet0/0
                     [110/20] via 10.1.1.158, 00:07:03, GigabitEthernet0/1
O E2    10.1.1.128/28 [110/20] via 10.1.1.154, 00:07:03, GigabitEthernet0/0
                      [110/20] via 10.1.1.158, 00:07:03, GigabitEthernet0/1
```

`O E2` indica uma rota externa do OSPF, do tipo 2. Ela não foi aprendida dentro do OSPF: veio do RIP e foi injetada por R-BORDA com o `redistribute rip subnets`. O `110` é a distância administrativa do OSPF. O `20` é a métrica-semente padrão que o IOS atribui quando o `redistribute` do OSPF não define métrica.

Em MTZ-R3, as mesmas redes aparecem com o mesmo valor:

```text
MTZ-R3#show ip route
O E2    10.1.1.96/27 [110/20] via 10.1.1.170, 00:32:22, Serial0/3/0
O E2    10.1.1.128/28 [110/20] via 10.1.1.170, 00:32:22, Serial0/3/0
```

MTZ-R3 está ligado direto a R-BORDA, e MTZ-R1 está um enlace mais longe. Mesmo assim, os dois mostram 20. Na rota E2, a métrica não acumula o custo interno do caminho: ela continua igual em todos os roteadores do domínio OSPF.

Evidências: `evidencias/etapa3.pdf`, seção 2.1, e `evidencias/configs/etapa3_MTZ-R3.txt` (configuração de MTZ-R3); saída de MTZ-R3 colada acima.

### Q14. Rotas de segunda mão no RIP

A LAN-SRV aparece em FIL-R2 como rota RIP com métrica 4:

```text
FIL-R2#show ip route
R       10.1.1.144/29 [120/4] via 10.1.1.177, 00:00:24, Serial0/3/0
```

Em FIL-R1, a mesma rota tem métrica 3:

```text
FIL-R1#show ip route
R       10.1.1.144/29 [120/3] via 10.1.1.173, 00:00:15, Serial0/3/1
```

A métrica 4 é a soma da métrica-semente 3, definida em R-BORDA, com 1 salto dentro do domínio RIP (de FIL-R1 até FIL-R2). A semente já ocupa 3 dos 15 saltos válidos do RIP.

Se houvesse mais 13 roteadores em cadeia depois de FIL-R2, cada um somaria 1 à métrica. O 12º roteador já receberia a rota com métrica 16, que significa inalcançável. A partir dele, a LAN-SRV deixaria de existir na tabela, e esses roteadores perderiam acesso ao servidor.

Evidências: `evidencias/etapa3.pdf`, seções 2.2 e 3.2.

### Q15. A jornada completa de um DISCOVER

O PC-FIL2-1 foi convertido para DHCP e a obtenção de endereço foi capturada no modo Simulation.

Caminho do DISCOVER até o servidor:

| Trecho | Rota usada | Aprendida por |
|---|---|---|
| PC-FIL2-1 → SW-FIL2 → FIL-R2 | broadcast na LAN; FIL-R2 aplica o relay e envia em unicast para 10.1.1.146 | - |
| FIL-R2 → FIL-R1 | `R 10.1.1.144/29 [120/4]` | RIP (rota redistribuída do OSPF) |
| FIL-R1 → R-BORDA | `R 10.1.1.144/29 [120/3]` | RIP (rota redistribuída do OSPF) |
| R-BORDA → MTZ-R3 | `O 10.1.1.144/29 [110/66]`, uma das duas rotas de custo igual | OSPF |
| MTZ-R3 → MTZ-R1 | `O 10.1.1.144/29 [110/2]` | OSPF |
| MTZ-R1 → SW-SRV → SRV-DC | `C 10.1.1.144/29` | rede conectada |

Caminho de volta do OFFER, endereçado ao GIADDR 10.1.1.129, da LAN-FIL2:

| Trecho | Rota usada | Aprendida por |
|---|---|---|
| SRV-DC → SW-SRV → MTZ-R1 | gateway padrão do servidor (10.1.1.145) | configuração estática do servidor |
| MTZ-R1 → MTZ-R2 | `O E2 10.1.1.128/28 [110/20]`, uma das duas rotas de custo igual | OSPF externa (redistribuída do RIP) |
| MTZ-R2 → R-BORDA | `O E2 10.1.1.128/28 [110/20]` | OSPF externa (redistribuída do RIP) |
| R-BORDA → FIL-R1 | `R 10.1.1.128/28 [120/2]` | RIP |
| FIL-R1 → FIL-R2 | `R 10.1.1.128/28 [120/1]` | RIP |
| FIL-R2 → SW-FIL2 → PC-FIL2-1 | `C 10.1.1.128/28` | rede conectada |

A ida passou por MTZ-R3 e a volta por MTZ-R2. Isso acontece porque R-BORDA e MTZ-R1 têm duas rotas de custo igual para o destino, e cada sentido escolheu um dos caminhos.

O roteador não consulta o protocolo para encaminhar o pacote. Ele consulta a tabela de rotas. O RIP e o OSPF só preenchem essa tabela.

Cinco roteadores separam o cliente do servidor: FIL-R2, FIL-R1, R-BORDA, MTZ-R3 (ou MTZ-R2) e MTZ-R1. Isso não impede o serviço porque só o primeiro trecho é broadcast. O relay de FIL-R2 transforma o DISCOVER em um pacote unicast comum, que é roteado como qualquer outro. Basta que exista rota nos dois sentidos, e a redistribuição garante isso.

Evidências: `evidencias/prints/q15_discover_jornada.png`, `evidencias/prints/q15_offer_retorno.png` e as tabelas das seções 2.1, 2.2 e 2.3 de `evidencias/etapa3.pdf`.

### Q16. Falha com rede viva

R-BORDA tem duas rotas de custo igual para a LAN-SRV:

```text
R-BORDA#show ip route
O       10.1.1.144/29 [110/66] via 10.1.1.165, 00:02:24, Serial0/0/0
                      [110/66] via 10.1.1.169, 00:02:24, Serial0/0/1
```

As duas existem porque os dois caminhos têm o mesmo custo. Por MTZ-R2: serial (64) + enlace GigE até MTZ-R1 (1) + LAN-SRV (1) = 66. Por MTZ-R3, a conta é a mesma. O OSPF instala as duas e divide o tráfego entre elas (ECMP).

No `tracert` antes da falha, o 4º salto foi `10.1.1.169`, ou seja, o caminho saiu por `Serial0/0/1`, para MTZ-R3:

```text
C:\>tracert 10.1.1.146
  1   0 ms      0 ms      0 ms      10.1.1.129
  2   1 ms      5 ms      4 ms      10.1.1.177
  3   4 ms      1 ms      4 ms      10.1.1.173
  4   6 ms      6 ms      10 ms     10.1.1.169
  5   0 ms      1 ms      3 ms      10.1.1.153
  6   6 ms      2 ms      3 ms      10.1.1.146
```

Com `shutdown` na `Serial0/0/1`, o caminho migrou para a `Serial0/0/0`, por MTZ-R2 (`10.1.1.165`), sem perda de conectividade:

```text
C:\>tracert 10.1.1.146
  1   0 ms      0 ms      0 ms      10.1.1.129
  2   2 ms      2 ms      2 ms      10.1.1.177
  3   5 ms      1 ms      6 ms      10.1.1.173
  4   1 ms      1 ms      9 ms      10.1.1.165
  5   1 ms      10 ms     7 ms      10.1.1.153
  6   2 ms      1 ms      0 ms      10.1.1.146
```

Derrubar a serial que não aparecia no primeiro `tracert` não mudaria nada visível. O ECMP divide o tráfego por fluxo, não por pacote: todos os pacotes de um mesmo par origem/destino seguem o mesmo caminho. Retirar o caminho que o fluxo não usava deixa o fluxo onde estava, embora a tabela perca uma das rotas.

Com a serial ainda derrubada, o PC-FIL2-1 renovou o endereço normalmente:

```text
C:\>ipconfig /release
   IP Address......................: 0.0.0.0
   Subnet Mask.....................: 0.0.0.0
   Default Gateway.................: 0.0.0.0
   DNS Server......................: 0.0.0.0

C:\>ipconfig /renew
   IP Address......................: 10.1.1.132
   Subnet Mask.....................: 255.255.255.240
   Default Gateway.................: 10.1.1.129
   DNS Server......................: 0.0.0.0
```

Dessa vez, o DISCOVER e o OFFER passaram pela única serial restante, entre R-BORDA e MTZ-R2. O serviço sobreviveu porque o DHCP centralizado depende só de existir rota entre o relay e o servidor. Quando a serial caiu, o OSPF retirou o caminho morto e manteve o outro, que já estava na tabela. Depois do `no shutdown`, as duas rotas voltaram.

Evidências: `evidencias/etapa3.pdf`, seções 2.3, 5.1, 5.2, 5.3 e 5.4; saída do `ipconfig /renew` colada acima.

### Q17. A fronteira não é simétrica

As três linhas observadas:

| Observação | Linha |
|---|---|
| /30 entre R-BORDA e FIL-R1, em MTZ-R1 | `O E2 10.1.1.172/30 [110/20] via 10.1.1.154` e `via 10.1.1.158` |
| /30 do triângulo, em FIL-R2 | `R 10.1.1.152/30 [120/4] via 10.1.1.177` |
| /30 entre R-BORDA e a matriz, em FIL-R2 | `R 10.1.1.164/30 [120/2] via 10.1.1.177` |

Em R-BORDA, os três /30 de fronteira aparecem como redes conectadas:

```text
R-BORDA#show ip route
C       10.1.1.164/30 is directly connected, Serial0/0/0
C       10.1.1.168/30 is directly connected, Serial0/0/1
C       10.1.1.172/30 is directly connected, Serial0/1/0
```

Três mecanismos explicam as observações.

**O que a redistribuição carrega.** O `redistribute ospf 1 metric 3` leva para o RIP as rotas que R-BORDA aprendeu pelo OSPF. Por isso, os /30 do triângulo chegam a FIL-R2 como rota redistribuída: semente 3 + 1 salto = 4. Os dois /30 entre R-BORDA e a matriz não são rotas aprendidas pelo OSPF em R-BORDA: são redes conectadas (código C). Por isso, eles não chegam à filial pela redistribuição.

**O que o `network 10.0.0.0` coloca no RIP.** Por ser classful, o comando coloca no processo RIP todas as interfaces 10.x de R-BORDA, incluindo as duas seriais da matriz. As redes dessas interfaces passam a ser anunciadas pelo RIP como redes diretamente conectadas, com métrica 1. Elas chegam a FIL-R1 com métrica 1 e a FIL-R2 com métrica 2, valor menor do que o das LANs da matriz (4), que dependem da semente 3.

**O que o `passive-interface` suprime.** As seriais da matriz são passivas no RIP: R-BORDA não envia updates por elas. A rede dessas interfaces, porém, continua dentro do processo e continua sendo anunciada pela interface ativa, para a filial. O passive impede o envio, mas não retira a rede do RIP.

O enunciado previa que o /30 entre R-BORDA e FIL-R1 não apareceria em MTZ-R1, porque é uma rede conectada de R-BORDA e não está em nenhum `network` do OSPF. No Packet Tracer 9.0, ela apareceu como `O E2 [110/20]`. Como essa rede está dentro do processo RIP (pelo `network 10.0.0.0`), o `redistribute rip subnets` a levou junto para o OSPF. O mesmo vale para os outros dois /30 de fronteira, mas eles não aparecem como E2 em MTZ-R1: já são conhecidos dentro do OSPF como rotas internas (`O 10.1.1.164/30 [110/65]`), que têm preferência sobre as externas.

Evidências: `evidencias/etapa3.pdf`, seções 1.1, 2.1, 2.2 e 2.4.

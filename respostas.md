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

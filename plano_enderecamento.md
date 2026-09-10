# Plano de Endereçamento IP

Bloco escolhido: **10.1.1.0/24**

O bloco foi dividido com VLSM, começando pelas redes com mais hosts. O primeiro IP útil de cada LAN ficou reservado para o gateway.

| Sub-rede | Hosts necessários | Rede e prefixo | Máscara | Faixa útil | Broadcast | Endereços usados | Justificativa |
|---|---:|---|---|---|---|---|---|
| LAN-ADM | 60 | 10.1.1.0/26 | 255.255.255.192 | 10.1.1.1 até 10.1.1.62 | 10.1.1.63 | MTZ-R3: 10.1.1.1. DHCP a partir de 10.1.1.2 | /26 possui 62 IPs úteis e atende 60 hosts. |
| LAN-ENG | 25 | 10.1.1.64/27 | 255.255.255.224 | 10.1.1.65 até 10.1.1.94 | 10.1.1.95 | MTZ-R2: 10.1.1.65. DHCP a partir de 10.1.1.66 | /27 possui 30 IPs úteis e atende 25 hosts. |
| LAN-FIL1 | 20 | 10.1.1.96/27 | 255.255.255.224 | 10.1.1.97 até 10.1.1.126 | 10.1.1.127 | FIL-R1: 10.1.1.97. DHCP a partir de 10.1.1.98 | /27 possui 30 IPs úteis e atende 20 hosts. |
| LAN-FIL2 | 10 | 10.1.1.128/28 | 255.255.255.240 | 10.1.1.129 até 10.1.1.142 | 10.1.1.143 | FIL-R2: 10.1.1.129. DHCP a partir de 10.1.1.130 | /28 possui 14 IPs úteis e atende 10 hosts. |
| LAN-SRV | 5 | 10.1.1.144/29 | 255.255.255.248 | 10.1.1.145 até 10.1.1.150 | 10.1.1.151 | MTZ-R1: 10.1.1.145. SRV-DC: 10.1.1.146 | /29 possui 6 IPs úteis e atende a rede dos servidores. |
| MTZ-R1 - MTZ-R2 | 2 | 10.1.1.152/30 | 255.255.255.252 | 10.1.1.153 até 10.1.1.154 | 10.1.1.155 | MTZ-R1: 10.1.1.153. MTZ-R2: 10.1.1.154 | /30 possui os dois IPs úteis necessários. |
| MTZ-R1 - MTZ-R3 | 2 | 10.1.1.156/30 | 255.255.255.252 | 10.1.1.157 até 10.1.1.158 | 10.1.1.159 | MTZ-R1: 10.1.1.157. MTZ-R3: 10.1.1.158 | /30 possui os dois IPs úteis necessários. |
| MTZ-R2 - MTZ-R3 | 2 | 10.1.1.160/30 | 255.255.255.252 | 10.1.1.161 até 10.1.1.162 | 10.1.1.163 | MTZ-R2: 10.1.1.161. MTZ-R3: 10.1.1.162 | /30 possui os dois IPs úteis necessários. |
| MTZ-R2 - R-BORDA | 2 | 10.1.1.164/30 | 255.255.255.252 | 10.1.1.165 até 10.1.1.166 | 10.1.1.167 | MTZ-R2: 10.1.1.165. R-BORDA: 10.1.1.166 | /30 possui os dois IPs úteis necessários. |
| MTZ-R3 - R-BORDA | 2 | 10.1.1.168/30 | 255.255.255.252 | 10.1.1.169 até 10.1.1.170 | 10.1.1.171 | MTZ-R3: 10.1.1.169. R-BORDA: 10.1.1.170 | /30 possui os dois IPs úteis necessários. |
| R-BORDA - FIL-R1 | 2 | 10.1.1.172/30 | 255.255.255.252 | 10.1.1.173 até 10.1.1.174 | 10.1.1.175 | R-BORDA: 10.1.1.173. FIL-R1: 10.1.1.174 | /30 possui os dois IPs úteis necessários. |
| FIL-R1 - FIL-R2 | 2 | 10.1.1.176/30 | 255.255.255.252 | 10.1.1.177 até 10.1.1.178 | 10.1.1.179 | FIL-R1: 10.1.1.177. FIL-R2: 10.1.1.178 | /30 possui os dois IPs úteis necessários. |

Os endereços de 10.1.1.180 até 10.1.1.255 não foram usados na topologia.

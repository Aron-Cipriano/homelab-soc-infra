# Write-up 01: Detecção de resposta de comando com privilégio de root

## Objetivo
Validar se o Suricata na vm-agente detecta, na rede Host-Only, um padrão de pós-exploração: a saída do comando `id` mostrando acesso de root.

## Ambiente
- Sensor: Suricata 8.0.3 na vm-agente (192.168.56.103), modo IDS, AF_PACKET na `enp0s8`
- Regras: ET Open, 53.383 ativas (`suricata-update`)
- HOME_NET: 192.168.56.0/24
- Origem simulada: vm-zabbix (192.168.56.101)

## Procedimento
1. Na vm-zabbix, um servidor HTTP do Python serviu um arquivo com o texto `uid=0(root) gid=0(root) groups=0(root)`.
2. Na vm-agente, `curl http://192.168.56.101:8080/id.txt`.
3. Consulta dos alertas no `eve.json` com `jq`.

O teste padrão (`curl testmynids.org`) não foi usado porque esse tráfego sai pela placa NAT, que o sensor não monitora.

## Eventos observados
| Horário | Origem → destino | Assinatura | SID |
|---|---|---|---|
| 16:29:11 | 192.168.56.101 → 192.168.56.103 | ET INFO Python SimpleHTTP ServerBanner | 2034636 |
| 16:29:11 | 192.168.56.101 → 192.168.56.103 | GPL ATTACK_RESPONSE id check returned root | 2100498 |

## Interpretação
- **SID 2100498:** a carga da resposta continha a saída do `id` com uid 0. Num ambiente real, indica que um invasor executou comandos com privilégio de root e está confirmando o acesso.
- **SID 2034636:** regra informativa; identifica um servidor HTTP improvisado com Python. Não é ataque por si só, mas é incomum em servidores de produção.

## Possíveis falsos positivos
- 2100498: documentação, tutoriais ou logs que contenham a saída do `id` trafegando em texto puro.
- 2034636: desenvolvedores usando `python3 -m http.server` para compartilhar arquivos.

## MITRE ATT&CK
- T1033 – System Owner/User Discovery: o comando `id` é usado para descobrir o usuário e os privilégios obtidos.

## Conclusão
O sensor está posicionado corretamente e detecta conteúdo malicioso na Host-Only em tempo real. A triagem separou um alerta de ameaça de um alerta informativo, que exigiria apenas verificação de contexto.
# Write-up 02: Varredura de rede interna e o papel do EXTERNAL_NET

## Objetivo
Verificar se o Suricata detecta varreduras de rede feitas de dentro da rede Host-Only e entender o que as regras ET Open cobrem com a configuração padrão.

## Ambiente
- Sensor: Suricata 8.0.3 na vm-agente (192.168.56.103), escutando a `enp0s8`
- Regras: ET Open, 53.383 ativas
- Atacante: vm-zabbix (192.168.56.101), Nmap 7.98

## Procedimento
Três varreduras da vm-zabbix contra a vm-agente, executadas duas vezes:
1. `nmap -sS -T4` (SYN scan, 1000 portas)
2. `nmap -sX -p 1-100` (Xmas scan)
3. `nmap -O` (detecção de sistema operacional)

- **Teste A:** configuração padrão, `EXTERNAL_NET: "!$HOME_NET"`
- **Teste B:** `EXTERNAL_NET: "any"`

Uma tentativa anterior, com o nmap rodando na própria vm-agente, não gerou nenhum evento: o tráfego para o próprio IP não passa pela placa de rede monitorada (`Network Distance: 0 hops`).

## Eventos observados

| Alerta | SID | Teste A | Teste B |
|---|---|---|---|
| SURICATA ICMPv4 unknown code | 2200025 | 10 | 10 |
| ET SCAN Potential SSH Scan OUTBOUND | 2003068 | 0 | 7 |
| ET SCAN NMAP OS Detection Probe | 2018489 | 0 | 5 |
| ET SCAN Suspicious inbound to PostgreSQL / MySQL / Oracle / MSSQL | 2010935–2010939 | 0 | 8 |
| ET SCAN Potential VNC Scan 5800-5820 | 2002910 | 0 | 1 |
| ET SCAN Potential SSH Scan | 2001219 | 0 | 1 |
| Xmas scan (alerta específico) | — | 0 | 0 |

## Interpretação
- No Teste A, o Suricata registrou as conexões da varredura (eventos `flow`), mas não alertou: as regras de varredura do ET são escritas como `$EXTERNAL_NET → $HOME_NET`, voltadas para ataques vindos de fora. Uma varredura interna não se encaixa nelas.
- O único alerta do Teste A veio de uma regra de anomalia de protocolo, que não depende de `HOME_NET` e `EXTERNAL_NET`: as sondagens ICMP com código inválido usadas pelo `-O`. O campo `CD=S` no fingerprint do nmap confirma que o Linux devolve o mesmo código ICMP recebido.
- No Teste B, as regras passaram a valer para tráfego interno e detectaram a varredura de portas e a detecção de sistema.
- Efeito colateral: 7 alertas "OUTBOUND" para uma varredura que entrou na vm-agente. Com `"any"`, as regras perdem a noção de direção.
- A varredura Xmas não gerou alerta específico em nenhum dos testes.

## Possíveis falsos positivos
- Ferramentas legítimas de inventário ou de varredura de vulnerabilidades na rede interna.
- Monitoramento que testa portas de serviços (por exemplo, checagens de disponibilidade).

## MITRE ATT&CK
- T1046 – Network Service Discovery: varredura de portas e identificação de serviços e do sistema operacional a partir de uma máquina interna.

## Conclusão
Visibilidade não é detecção. Com a configuração padrão, uma varredura interna, típica de movimentação lateral, passa sem alerta de varredura. Para o laboratório, `EXTERNAL_NET` foi mantido como `"any"`, aceitando mais ruído e alertas com direção errada. A alternativa mais adequada para produção é manter o padrão e criar regras próprias para reconhecimento interno, o que será feito numa etapa futura. Com o firewall segmentando a rede (fase futura), um atacante externo real deixará de depender desse ajuste.

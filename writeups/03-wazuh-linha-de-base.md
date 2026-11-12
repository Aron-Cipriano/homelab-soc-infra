# Write-up 03: Primeira triagem no SIEM e a linha de base do ambiente

## Objetivo
Analisar os primeiros eventos registrados pelo Wazuh logo após a instalação, separar o que é atividade administrativa legítima do que seria suspeito e estabelecer a linha de base (o comportamento normal) do laboratório.

## Ambiente
- SIEM: Wazuh 4.14.10 all-in-one na vm-wazuh (192.168.56.110)
- Agentes: vm-wazuh (o próprio Manager monitora o host) e vm-agente (192.168.56.103, com Suricata)
- Período analisado: 12/11/2026, das 14h00 às 15h45, cobrindo a instalação do Wazuh, a instalação do agente e os testes do Suricata

## Visão geral (Threat Hunting, últimas 24 h)

| Indicador | Valor |
|---|---|
| Eventos no total | 454 |
| Alertas de nível 12 ou mais | 0 |
| Falhas de autenticação | 0 |
| Autenticações com sucesso | 30 |
| Níveis observados | 3, 4, 5, 7 e 8 |
| Agente com mais eventos | vm-wazuh |

Técnicas MITRE ATT&CK mapeadas pelo Wazuh: Valid Accounts (T1078), Sudo and Sudo Caching (T1548.003), Create Account (T1136) e Disable or Modify Tools (T1562.001).

O gráfico de evolução mostra um pico perto das 14h, que coincide com a instalação do Wazuh.

## Eventos analisados

Regras que mais dispararam no período:

| Ocorrências | Regra | Nível | Agente | Descrição do Wazuh | MITRE |
|---|---|---|---|---|---|
| 56 | 510 | 7 | vm-agente | Host-based anomaly detection event (rootcheck) | — |
| 26 / 11 | 5502 | 3 | vm-wazuh / vm-agente | PAM: Login session closed | — |
| 24 / 7 | 5501 | 3 | vm-wazuh / vm-agente | PAM: Login session opened | T1078 |
| 20 / 6 | 5402 | 3 | vm-wazuh / vm-agente | Successful sudo to ROOT executed | T1548.003 |
| 4 | 502 | 3 | vm-wazuh | Wazuh server started | — |
| 9 | 86601 | 3 | vm-agente | Suricata: Alert (5 assinaturas diferentes) | — |
| 2 | 506 | 3 | vm-agente | Wazuh agent stopped | T1562.001 |
| 2 | 19003 | 5 | vm-wazuh | SCA: CIS Ubuntu Linux 24.04 Benchmark, nota 50% | — |
| 1 | 5902 | 8 | vm-wazuh | New user added to the system | T1136 |

O alerta mais frequente e de nível mais alto (regra 510, nível 7) não era atividade administrativa: foi o primeiro item da triagem.

## Interpretação

### 0. Rootcheck: "Trojaned version of file" na vm-agente (falso positivo confirmado)
- **O que mostra:** 56 alertas de nível 7 apontando versões "trojanizadas" de `ls`, `cat`, `chmod`, `chown`, `chgrp`, `echo`, `env` e `uname` na vm-agente.
- **Como a regra decide:** a assinatura é genérica (`Generic`). Ela procura dentro do binário textos como `bash`, `/bin/sh`, `proc.h` e `file.h`, típicos de versões adulteradas antigas. Ela não compara o arquivo com o original.
- **Investigação:**
  - Na vm-agente (Ubuntu 26.04), `readlink -f /usr/bin/ls` aponta para `/usr/lib/cargo/bin/coreutils/ls`, do pacote `rust-coreutils`: uma reimplementação dos comandos básicos em Rust, em que vários comandos compartilham o mesmo binário e seus textos.
  - Na vm-wazuh (Ubuntu 24.04), o `ls` é um arquivo comum do pacote `coreutils` e não gera o alerta. Serviu de comparação.
  - `dpkg --verify rust-coreutils` não apontou nenhum arquivo diferente dos hashes do pacote.
- **Classificação:** falso positivo. Os arquivos são os originais do pacote; a assinatura genérica não reconhece o novo formato dos binários.
- **Tratamento:** desativada apenas a checagem de trojans do rootcheck na vm-agente (`check_trojans`). A integridade dos binários continua coberta pelo monitoramento de arquivos (FIM) do Wazuh, que alerta qualquer alteração em `/usr/bin`, e pela verificação de pacotes.

### 1. Login com conta válida (T1078)
- **O que mostra:** login SSH do usuário `vboxuser` a partir de `192.168.56.1`, o host Windows do administrador.
- **Por que é benigno:** origem conhecida, usuário conhecido, horário de trabalho no laboratório.
- **O que o tornaria suspeito:** origem desconhecida, login fora do horário habitual, muitas falhas antes do sucesso (possível força bruta) ou um usuário que normalmente não acessa por SSH.

### 2. Uso de sudo (T1548.003)
- **O que mostra:** comandos executados com privilégio de administrador durante a instalação e a configuração.
- **Por que é benigno:** cada comando corresponde a uma etapa documentada da instalação.
- **O que o tornaria suspeito:** `sudo` por um usuário que não é administrador, comandos para ler arquivos de senha ou desligar serviços de segurança, ou uso logo após um login de origem estranha.

### 3. Criação de usuário (T1136)
- **O que mostra:** regra 5902, às 14:18:26: `useradd ... name=wazuh-dashboard, UID=111, home=/nonexistent, shell=/bin/false`, criado pela instalação do Wazuh.
- **Por que é benigno:** o horário coincide com a instalação, o nome corresponde ao serviço instalado, e a conta não tem pasta pessoal nem shell (`/bin/false`), ou seja, ninguém consegue fazer login com ela.
- **O que o tornaria suspeito:** usuário criado sem mudança programada, com nome imitando contas legítimas, adicionado ao grupo `sudo` ou com shell de login em vez de um usuário de serviço.

### 4. Desativação ou alteração de ferramentas (T1562.001)
Corresponde às paradas e reinícios do agente e do Manager durante a configuração. Num incidente real, é o que um invasor faria para cegar o monitoramento; aqui, cada parada está ligada a uma ação documentada.

## Possíveis falsos positivos
O rootcheck da vm-agente foi um falso positivo (o alerta descreve algo que não aconteceu). Todos os outros eventos do período são verdadeiros (aconteceram), mas benignos (administração legítima). A diferença entre "verdadeiro positivo" e "verdadeiro benigno" depende do contexto: quem fez, de onde, quando e se havia mudança planejada.

## Conclusão
A triagem começou pelo alerta mais frequente e de nível mais alto, e ele se mostrou um falso positivo causado por uma mudança no sistema operacional, confirmado com verificação de integridade dos pacotes. Os demais eventos do SIEM são, em grande parte, o próprio administrador trabalhando. Registrar essa linha de base permite reconhecer o anormal depois: o mesmo alerta de criação de usuário às 3h da manhã, sem mudança programada, seria investigado. Um ponto observado: o Wazuh mapeia técnicas MITRE para eventos de sistema, mas não para os alertas do Suricata, que chegam todos com nível 3. A priorização desses alertas será feita com regras locais.

## Como reproduzir a análise
Resumo das regras disparadas, direto no Manager:

```bash
sudo jq -r '[.rule.id, .rule.level, .agent.name, .rule.description, ((.rule.mitre.id // []) | join(","))] | @tsv' /var/ossec/logs/alerts/alerts.json | sort | uniq -c | sort -rn | head -20
```

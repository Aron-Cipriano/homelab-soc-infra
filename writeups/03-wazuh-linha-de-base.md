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

| # | Horário | Agente | Regra (ID e nível) | Descrição do Wazuh | MITRE | Classificação |
|---|---|---|---|---|---|---|
| 1 | _preencher_ | _preencher_ | _preencher_ | Login SSH com sucesso | T1078 | Verdadeiro, benigno |
| 2 | _preencher_ | _preencher_ | _preencher_ | Uso de `sudo` | T1548.003 | Verdadeiro, benigno |
| 3 | _preencher_ | _preencher_ | _preencher_ | Criação de usuário | T1136 | Verdadeiro, benigno |

## Interpretação

### 1. Login com conta válida (T1078)
- **O que mostra:** login SSH do usuário `vboxuser` a partir de `192.168.56.1`, o host Windows do administrador.
- **Por que é benigno:** origem conhecida, usuário conhecido, horário de trabalho no laboratório.
- **O que o tornaria suspeito:** origem desconhecida, login fora do horário habitual, muitas falhas antes do sucesso (possível força bruta) ou um usuário que normalmente não acessa por SSH.

### 2. Uso de sudo (T1548.003)
- **O que mostra:** comandos executados com privilégio de administrador durante a instalação e a configuração.
- **Por que é benigno:** cada comando corresponde a uma etapa documentada da instalação.
- **O que o tornaria suspeito:** `sudo` por um usuário que não é administrador, comandos para ler arquivos de senha ou desligar serviços de segurança, ou uso logo após um login de origem estranha.

### 3. Criação de usuário (T1136)
- **O que mostra:** criação de usuários de sistema pelos pacotes instalados (por exemplo, o usuário do agente Wazuh).
- **Por que é benigno:** o horário coincide com a instalação dos pacotes, e o nome do usuário corresponde ao serviço instalado.
- **O que o tornaria suspeito:** usuário criado sem mudança programada, com nome imitando contas legítimas, adicionado ao grupo `sudo` ou com shell de login em vez de um usuário de serviço.

### 4. Desativação ou alteração de ferramentas (T1562.001)
Corresponde às paradas e reinícios do agente e do Manager durante a configuração. Num incidente real, é o que um invasor faria para cegar o monitoramento; aqui, cada parada está ligada a uma ação documentada.

## Possíveis falsos positivos
Todos os eventos do período são verdadeiros (aconteceram), mas benignos (administração legítima). A diferença entre "verdadeiro positivo" e "verdadeiro benigno" depende do contexto: quem fez, de onde, quando e se havia mudança planejada.

## Conclusão
Os primeiros eventos do SIEM são, em grande parte, o próprio administrador trabalhando. Registrar essa linha de base permite reconhecer o anormal depois: o mesmo alerta de criação de usuário às 3h da manhã, sem mudança programada, seria investigado. Um ponto observado: o Wazuh mapeia técnicas MITRE para eventos de sistema, mas não para os alertas do Suricata, que chegam todos com nível 3. A priorização desses alertas será feita com regras locais.

## Como reproduzir a análise
Resumo das regras disparadas, direto no Manager:

```bash
sudo jq -r '[.rule.id, .rule.level, .agent.name, .rule.description, ((.rule.mitre.id // []) | join(","))] | @tsv' /var/ossec/logs/alerts/alerts.json | sort | uniq -c | sort -rn | head -20
```

# Write-up 04: Regra local no Wazuh para priorizar um alerta do Suricata

## Objetivo
Corrigir a priorização de um alerta de rede que indica possível pós-exploração. No Wazuh, todos os alertas do Suricata chegavam com o mesmo nível baixo e sem mapeamento no MITRE ATT&CK.

## Ambiente
- SIEM: Wazuh 4.14.10 na vm-wazuh (192.168.56.110)
- Sensor: Suricata 8.0.3 na vm-agente (192.168.56.103), com Wazuh Agent lendo o `eve.json`
- Regra padrão do Wazuh para alertas do Suricata: 86601, nível 3

## Problema observado
- Todos os alertas do Suricata recebiam nível 3, sem distinção de gravidade. O alerta "id check returned root" ficava com a mesma prioridade do "servidor HTTP do Python detectado", que é só informativo.
- Nenhum alerta do Suricata aparecia no painel MITRE ATT&CK do Wazuh.
- O próprio Suricata classifica a assinatura 2100498 como `signature_severity: Informational` e `confidence: Medium`. A decisão de priorizar é do analista, pelo contexto do ambiente: neste laboratório, root aparecendo no tráfego indica possível pós-exploração.

## A regra

Arquivo no Manager: `/var/ossec/etc/rules/local_rules.xml` (versionado em [`wazuh/rules/local_rules.xml`](../wazuh/rules/local_rules.xml)).

```xml
<group name="local,suricata,">
  <rule id="100100" level="10">
    <if_sid>86601</if_sid>
    <field name="alert.signature_id">^2100498$|^2019284$</field>
    <description>Suricata: saida do comando id com privilegio de root no trafego (possivel pos-exploracao) - $(alert.signature)</description>
    <mitre>
      <id>T1033</id>
    </mitre>
    <group>pos_exploracao,</group>
  </rule>
</group>
```

- `if_sid 86601`: a regra só avalia eventos que já foram classificados como alerta do Suricata.
- `alert.signature_id`: compara pelo ID das assinaturas 2100498 (GPL ATTACK_RESPONSE id check returned root) e 2019284 (ET ATTACK_RESPONSE Output of id command from HTTP server). As duas indicam o mesmo comportamento.
- `id 100100`: a faixa de 100000 a 120000 é reservada para regras locais.
- `mitre T1033`: System Owner/User Discovery, tática Discovery.

## Procedimento de validação
1. `wazuh-analysisd -t` para validar a sintaxe sem reiniciar o Manager.
2. `wazuh-logtest` com uma linha real do `eve.json`, para simular o evento sem gerar o ataque de novo.
3. Reinício do `wazuh-manager` para carregar a regra.
4. Teste ao vivo: servidor HTTP na vm-wazuh servindo `uid=0(root) gid=0(root) groups=0(root)` e `curl` a partir da vm-agente.
5. Conferência no `alerts.json` do Manager e no Dashboard.

## Resultados

| Etapa | Resultado |
|---|---|
| Primeira versão (expressão regular no nome da assinatura) | Não disparou no `wazuh-logtest`; o evento continuou na 86601 |
| Segunda versão (comparação pelo ID da assinatura) | Disparou no `wazuh-logtest`: regra 100100, nível 10, T1033 |
| Teste ao vivo | Alerta 100100, nível 10, no `alerts.json` e no Dashboard |
| Painel MITRE ATT&CK | Passou a mostrar a técnica System Owner/User Discovery |

**Linha do tempo do teste ao vivo:**

| Horário | Etapa |
|---|---|
| 16:43:56.666 | Suricata detecta o tráfego na vm-agente (`data.timestamp`) |
| 16:43:57.697 | Wazuh gera o alerta 100100 (`timestamp`), cerca de 1 segundo depois |
| ~16:45 | Alerta visível no Dashboard, depois da ingestão pelo Filebeat e indexação |

O mesmo tráfego disparou três assinaturas no Suricata (2034636, 2019284 e 2100498). A regra cobre as duas que indicam o comando `id` com root; a do servidor Python continua informativa.

## Interpretação
- A comparação pelo ID é mais robusta que pelo nome: o ID não muda entre versões das regras do Suricata, e o nome pode mudar. A primeira versão não disparou, e a causa (regra não gravada ou comparação com expressão regular) não foi isolada.
- O documento do alerta reúne o que o sensor viu (`data.alert.*`, `full_log` com o evento original) e o que o SIEM concluiu (`rule.*`, `rule.mitre.*`). O `full_log` preserva a evidência original.
- O tempo até o alerta aparecer para o analista inclui a ingestão (cerca de 1 a 2 minutos), e não só o processamento da regra.

## Possíveis falsos positivos
- Documentação ou páginas de tutorial com a saída do comando `id` trafegando em HTTP sem criptografia.
- Ferramentas de administração que executam comandos remotos e devolvem a saída em texto puro.

## MITRE ATT&CK
- T1033 – System Owner/User Discovery (tática Discovery).

## Conclusão
Uma regra local de poucas linhas transformou um alerta genérico de nível 3 num alerta priorizado e mapeado no ATT&CK, sem alterar as regras originais do Wazuh. A validação em etapas (sintaxe, simulação com `wazuh-logtest` e teste ao vivo) permitiu achar e corrigir uma primeira versão que não funcionava antes de confiar na detecção.

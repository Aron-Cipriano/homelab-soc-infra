# Regras Sigma

As detecções do laboratório reescritas em [Sigma](https://sigmahq.io), o formato aberto de regras de detecção. Uma regra Sigma descreve **o que** detectar sem depender do SIEM; ferramentas como o `sigma-cli` (pySigma) convertem a regra para a linguagem de cada plataforma (Splunk, Elastic, OpenSearch, Microsoft Sentinel e outras).

| Arquivo | Detecção | Regra no Wazuh | MITRE |
|---|---|---|---|
| [`suricata_id_root_no_trafego.yml`](suricata_id_root_no_trafego.yml) | Saída do `id` com root no tráfego | 100100 | T1033 |
| [`suricata_varredura_de_rede.yml`](suricata_varredura_de_rede.yml) | Assinaturas de varredura do ET | 100101 | T1046 |
| [`correlacao_varredura_insistente.yml`](correlacao_varredura_insistente.yml) | 5 alertas de varredura do mesmo IP em 2 minutos | 100102 | T1046 |
| [`linux_alteracao_etc_hosts.yml`](linux_alteracao_etc_hosts.yml) | Alteração do `/etc/hosts` | 550 (FIM) | T1565.001 |
| [`linux_usuario_criado_com_shell.yml`](linux_usuario_criado_com_shell.yml) | Usuário criado com shell de login | 100103 | T1136.001 |

A regra de correlação usa o formato de correlação do Sigma (`type: event_count`, `group-by`, `timespan`) e referencia a regra de varredura pelo campo `name`.

## Validação

```bash
pip install sigma-cli
sigma check -x attacktag -x d3_fendtag -x namespace_tag sigma/
```

Resultado: 0 erros, 0 problemas de condição e 0 problemas de validação. Os validadores excluídos baixam listas da internet na hora de rodar e foram desligados por causa da rede do ambiente de teste; as tags MITRE usadas são válidas.

## Conversão para o Wazuh

O Wazuh Indexer é baseado no OpenSearch, então aceita consultas Lucene. No índice do Wazuh, os campos do Suricata ficam dentro de `data.`, o que é resolvido com um pipeline de processamento ([`pipelines/wazuh-suricata.yml`](pipelines/wazuh-suricata.yml)):

```yaml
transformations:
  - id: prefixo_data
    type: field_name_prefix
    prefix: "data."
    rule_conditions:
      - type: logsource
        product: suricata
```

Conversão com o backend Lucene (`pysigma-backend-elasticsearch`):

| Regra | Consulta gerada |
|---|---|
| Varredura de rede | `data.event_type:alert AND (data.alert.signature_id:(2001219 OR 2003068 OR 2002910 OR 2018489 OR 2010935 OR 2010936 OR 2010937 OR 2010939))` |
| `id` com root | `data.event_type:alert AND (data.alert.signature_id:(2100498 OR 2019284))` |

As consultas funcionam direto na busca do Wazuh Dashboard (Threat Hunting → Events), com a linguagem trocada de DQL para Lucene. É a forma de fazer threat hunting com a regra antes de transformá-la em alerta.

## Por que manter as duas versões
- **XML do Wazuh:** é o que gera o alerta no laboratório.
- **Sigma:** documenta a lógica de forma portátil. Se o SIEM mudar, a detecção é convertida, não reescrita do zero.

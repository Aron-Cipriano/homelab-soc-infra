# homelab-soc-infra

Home lab de monitoramento (Zabbix e Grafana) e detecção (Suricata e Wazuh) rodando em VMs leves no VirtualBox. O objetivo é praticar infraestrutura e operações de SOC em hardware modesto e documentar cada etapa, incluindo os problemas encontrados e como foram resolvidos.

![Dashboard do Grafana](Imagens/dashboard-antes.png)

## Ambiente

- **Host:** Windows, Ryzen 3 3200G (4 núcleos), 16 GB de RAM
- **Hipervisor:** Oracle VirtualBox 7.2.22
- **Sistema das VMs:** Ubuntu Server 26.04.1 LTS (vm-zabbix e vm-agente) e 24.04.5 LTS (vm-wazuh, versão suportada pelo Wazuh)
- **Armazenamento:** arquivos das VMs em um disco separado do sistema
- **Acesso:** SSH a partir do host

### Versões

| Componente | Versão | VM |
|---|---|---|
| Zabbix server e frontend | 7.0.32 | vm-zabbix |
| Zabbix Agent 2 | 7.0.x | vm-zabbix e vm-agente |
| MariaDB | 11.8.6 | vm-zabbix |
| Apache | 2.4.66 | vm-zabbix |
| PHP-FPM | 8.5 | vm-zabbix |
| Grafana | 13.3.0 | vm-zabbix |
| Plugin Zabbix para Grafana | 6.10.2 | vm-zabbix |
| Suricata | 8.0.3 | vm-agente |
| Wazuh (indexer, manager e dashboard) | 4.14.10 | vm-wazuh |
| Wazuh Agent | 4.14.10 | vm-agente |

## Topologia

```mermaid
flowchart LR
    NET["Internet via NAT"]
    HOST["Host Windows com VirtualBox"]
    ZBX["vm-zabbix 192.168.56.101 - Zabbix server e Grafana"]
    AGT["vm-agente 192.168.56.103 - Zabbix Agent 2, Suricata e Wazuh Agent"]
    WZH["vm-wazuh 192.168.56.110 - Wazuh indexer, manager e dashboard"]
    NET --- ZBX
    NET --- AGT
    NET --- WZH
    HOST ---|"Host-Only"| ZBX
    HOST ---|"Host-Only"| AGT
    HOST ---|"Host-Only"| WZH
    ZBX ---|"portas 10050 e 10051"| AGT
    AGT ---|"portas 1514 e 1515"| WZH
```

Se o diagrama acima não aparecer no seu leitor de Markdown, esta é a mesma topologia em texto:

```
Windows (Ryzen 3 3200G, 16 GB) + VirtualBox 7.2.22
 ├─ vm-zabbix  192.168.56.101  (4 GB, 2 vCPU)
 │    Apache + PHP-FPM → painel Zabbix (/zabbix)
 │    Zabbix server → MariaDB
 │    Grafana :3000 → API do Zabbix
 ├─ vm-agente  192.168.56.103  (3 GB, 2 vCPU)
 │    Zabbix Agent 2 :10050
 │    Suricata (enp0s8) → eve.json → Wazuh Agent
 └─ vm-wazuh   192.168.56.110  (6 GB, 2 vCPU, 50 GB)
      Wazuh indexer + manager :1514/1515 + dashboard :443

Redes: NAT (internet) + Host-Only 192.168.56.0/24 (lab e acesso do host)
IPs fixos: Windows .1 | vm-zabbix .101 | vm-agente .103 | vm-wazuh .110 (DHCP da Host-Only desligado)
```

Cada VM tem duas placas de rede: a primeira em NAT, para acesso à internet, e a segunda em Host-Only, para o tráfego do lab e o acesso do host.

Os endereços da Host-Only são fixos: o host Windows usa `192.168.56.1` e cada VM tem seu IP configurado via netplan em `/etc/netplan/60-hostonly.yaml`. O servidor DHCP da Host-Only fica desligado, para que nenhum endereço mude depois de um reboot.

```yaml
# /etc/netplan/60-hostonly.yaml (vm-zabbix)
network:
  version: 2
  ethernets:
    enp0s8:
      dhcp4: false
      addresses: [192.168.56.101/24]
```

## VMs

| VM | Função | Recursos | IP Host-Only |
|---|---|---|---|
| vm-zabbix | Servidor de monitoramento (Zabbix, MariaDB, Apache, Grafana) | 4 GB RAM, 2 vCPU | 192.168.56.101 |
| vm-agente | Máquina monitorada e sensor de rede (Zabbix Agent 2, Suricata, Wazuh Agent) | 3 GB RAM, 2 vCPU | 192.168.56.103 |
| vm-wazuh | SIEM (Wazuh all-in-one) | 6 GB RAM, 2 vCPU, 50 GB, 2 GB de swap | 192.168.56.110 |

Com 16 GB no host, a vm-zabbix fica desligada nas sessões de Wazuh.

## Como funciona

- **Coleta:** o servidor consulta o agente na porta 10050 (checagens passivas) e o agente envia dados ao servidor na 10051 (checagens ativas). Os dados ficam no MariaDB.
- **Painel do Zabbix:** servido pelo Apache com PHP-FPM em `/zabbix`.
- **Grafana:** consulta a API JSON-RPC do Zabbix (`/zabbix/api_jsonrpc.php`) pelo plugin e monta os dashboards na porta 3000.

## Grafana

### Autenticação

O Grafana acessa o Zabbix com um **usuário de serviço dedicado** (`grafana-user`), seguindo o princípio do menor privilégio:

- Role do tipo *User*, com acesso à API e sem acesso ao frontend
- Permissão somente de **leitura** nos host groups `Zabbix servers` e `Linux servers`
- Autenticação por **API token**, revogável a qualquer momento em *Users → API tokens*
- Token sem expiração por ser um lab isolado em rede Host-Only
- Nenhuma credencial versionada neste repositório

As senhas padrão do Admin do Zabbix e do admin do Grafana foram alteradas.

### Dashboard "Homelab - Visão Geral"

Arquivo: [`grafana/dashboards/homelab-visao-geral.json`](grafana/dashboards/homelab-visao-geral.json)

| Painel | Visualização | Dado do Zabbix |
|---|---|---|
| Disponibilidade Agente | Stat (UP/DOWN) | `Zabbix agent availability` |
| Problemas ativos | Zabbix Problems | Problemas e recuperações recentes |
| CPU | Time series | `CPU utilization` |
| Memória | Time series | `Memory utilization` |
| Disco | Gauge | `FS [/]: Space: Used, in %` |
| Rede | Time series | `Interface enp0s8: Bits received/sent` |

Todos os painéis usam a variável `$host`, que permite filtrar uma VM ou ver todas.

**Para importar:** Dashboards → New → Import → envie o JSON e selecione o data source do Zabbix.

## Teste de detecção: queda do agente

Simulação de uma falha na vm-agente para validar se o monitoramento detecta o problema e registra a recuperação.

**Procedimento**

```bash
# na vm-agente
sudo systemctl stop zabbix-agent2
# aguardar a detecção
sudo systemctl start zabbix-agent2
```

**Resultado**

| Etapa | O que aconteceu |
|---|---|
| Antes | As duas VMs em UP, sem problemas ativos |
| Falha | O card da vm-agente mudou para DOWN e o problema *Linux: Zabbix agent is not available* (severidade Average) apareceu no painel |
| Recuperação | Depois de religar o agente, o card voltou para UP e o problema ficou como RESOLVED |

| Métrica | Valor |
|---|---|
| Tempo até a detecção | cerca de 2 minutos |
| Duração do problema | cerca de 5 minutos |

| Antes | Durante | Depois |
|---|---|---|
| ![Antes](Imagens/dashboard-antes.png) | ![Incidente](Imagens/dashboard-incidente.png) | ![Recuperado](Imagens/dashboard-recuperado.png) |

O Zabbix não marca o agente como indisponível na primeira falha: ele faz novas tentativas antes, o que evita falsos positivos por uma oscilação momentânea.

## Detecção: Suricata e Wazuh

### Pipeline

```
tráfego na Host-Only → Suricata (vm-agente) → eve.json → Wazuh Agent → Wazuh Manager (regras locais) → Indexer → Dashboard
```

- **Suricata:** captura em AF_PACKET na `enp0s8`, regras ET Open (53.383 ativas), `HOME_NET` = `192.168.56.0/24`. O `EXTERNAL_NET` foi definido como `any` para que as regras de varredura valham também para tráfego interno do lab (ver [write-up 02](writeups/02-suricata-varredura-nmap.md)).
- **Wazuh Agent:** lê o `/var/log/suricata/eve.json` em formato JSON:
  ```xml
  <localfile>
    <log_format>json</log_format>
    <location>/var/log/suricata/eve.json</location>
  </localfile>
  ```
- **Wazuh Manager:** todo alerta do Suricata chega pela regra padrão 86601 (nível 3). As regras locais reclassificam os alertas que importam.

### Regras locais (detecção como código)

Arquivo: [`wazuh/rules/local_rules.xml`](wazuh/rules/local_rules.xml)

| Regra | Nível | Lógica | MITRE |
|---|---|---|---|
| 100100 | 10 | Assinaturas 2100498 e 2019284: saída do comando `id` com root no tráfego (possível pós-exploração) | T1033 |
| 100101 | 8 | 8 assinaturas de varredura do ET (SSH, VNC, bancos de dados, detecção de SO) | T1046 |
| 100102 | 12 | Correlação: 5 alertas da 100101 do mesmo `src_ip` em 120 segundos | T1046 |

As regras comparam pelo `alert.signature_id`. A comparação pelo nome da assinatura não disparou nos testes, e o ID é mais estável.

**Deploy:** o arquivo é editado no repositório e copiado para o Manager, nunca colado no terminal:

```powershell
scp D:\homelab-soc-infra\wazuh\rules\local_rules.xml wazuh:/tmp/local_rules.xml
```
```bash
sudo cp /tmp/local_rules.xml /var/ossec/etc/rules/local_rules.xml
sudo chown wazuh:wazuh /var/ossec/etc/rules/local_rules.xml
sudo chmod 660 /var/ossec/etc/rules/local_rules.xml
sudo /var/ossec/bin/wazuh-analysisd -t   # valida antes de reiniciar
sudo systemctl restart wazuh-manager
```

Cada regra é testada antes com `wazuh-logtest` e depois com tráfego real.

### Resultado da correlação

Uma varredura com Nmap a partir da vm-wazuh gerou 18 alertas da 100101 (nível 8) e 4 da 100102 (nível 12). O ajuste previsto é o atributo `ignore` na 100102, para gerar um único alerta por varredura.

| Alertas de nível 12 no Dashboard | Alertas da regra 100102 |
|---|---|
| ![Nível 12](Imagens/wazuh-nivel12.png) | ![Correlação 100102](Imagens/wazuh-correlacao-100102.png) |

### Write-ups

| # | Título |
|---|---|
| 02 | [Varredura de rede interna e o papel do EXTERNAL_NET](writeups/02-suricata-varredura-nmap.md) |
| 03 | [Primeira triagem no SIEM e a linha de base do ambiente](writeups/03-wazuh-linha-de-base.md) |
| 04 | [Regra local no Wazuh para priorizar um alerta do Suricata](writeups/04-wazuh-regra-local-100100.md) |
| 05 | [Detecção de varredura e regra de correlação no Wazuh](writeups/05-wazuh-correlacao-varredura.md) |

## Problemas resolvidos

### 1. Apache não subia
- **Sintoma:** o painel do Zabbix não abria.
- **Causa:** faltavam os módulos `proxy` e `proxy_fcgi`, usados pelo Apache para encaminhar as requisições ao PHP-FPM.
- **Correção:** `sudo a2enmod proxy proxy_fcgi` e reload do Apache.

### 2. Esquema do banco incompleto
- **Sintoma:** erros na inicialização do Zabbix server.
- **Causa:** o servidor de banco não estava instalado, e uma importação interrompida deixou o esquema pela metade.
- **Correção:** instalação do MariaDB, recriação do banco e nova importação completa do esquema.

### 3. Grafana travado em "Testing..."
- **Sintoma:** o *Save & test* do data source nunca concluía.
- **Investigação:** a API JSON-RPC respondia normalmente, então o problema não era conectividade, e sim autenticação.
- **Causa:** o Apache não repassava o cabeçalho `Authorization` ao PHP-FPM.
- **Correção:** em `/etc/apache2/apache2.conf`:
  ```apache
  SetEnvIfNoCase ^Authorization$ "(.+)" HTTP_AUTHORIZATION=$1
  ```
  seguido de `sudo apache2ctl configtest` e `sudo systemctl reload apache2`.
- **Aprendizado:** endpoint respondendo não significa autenticação funcionando. É preciso checar o repasse de cabeçalhos entre o servidor web e a aplicação.

### 4. Bloqueio de conta e credenciais divergentes
- **Sintoma:** o Grafana parava de autenticar e a conta era bloqueada por tentativas seguidas.
- **Causa:** o Grafana usava a conta Admin, e as senhas configuradas nos dois lados divergiram.
- **Correção:** criação de um usuário de serviço somente leitura e autenticação por API token. Trocar a senha do Admin não afeta mais a integração.

### 5. Host "sumido" no Grafana
- **Sintoma:** a variável `$host` listava só 1 dos 2 hosts.
- **Investigação:** um `host.get` direto na API, com o token do `grafana-user`, retornou os 2 hosts. Logo, a permissão no Zabbix estava correta e o problema estava no Grafana.
  ```bash
  read -s TOKEN
  curl -s -X POST http://127.0.0.1/zabbix/api_jsonrpc.php \
    -H "Content-Type: application/json-rpc" \
    -H "Authorization: Bearer $TOKEN" \
    -d '{"jsonrpc":"2.0","method":"host.get","params":{"output":["host","name"]},"id":1}'
  ```
- **Causa:** o cache do plugin guardava a resposta de antes da liberação do host group `Linux servers`.
- **Correção:** restart do `grafana-server` e Cache TTL do data source reduzido para 5 minutos.

### 6. Saturação de CPU
- **Sintoma:** lentidão no painel e uso de CPU perto de 70% na vm-zabbix.
- **Causa:** com 1 vCPU, o PHP-FPM saturava a VM.
- **Correção:** vm-zabbix com 2 vCPUs.

### 7. Configuração de rede amarrada ao MAC da VM base
- **Sintoma:** o netplan da vm-zabbix tinha `match: macaddress` com o MAC da VM base.
- **Causa:** o instalador amarra a placa ao MAC; o clone herdou essa configuração, que não correspondia mais à placa da VM. A placa NAT dos clones também manteve o MAC da VM base.
- **Correção:** remoção do `match`/`set-name`, deixando a configuração pelo nome da interface (`enp0s3`). A vm-zabbix recebeu um MAC novo com `VBoxManage modifyvm --macaddress1 auto`.

### 8. Conflito de IP entre as VMs
- **Sintoma:** a vm-zabbix parou de responder no `.101` depois de configurar o IP fixo.
- **Investigação:** o `60-hostonly.yaml` da vm-zabbix tinha `192.168.56.103`, o IP da vm-agente, e o `ip -br a` confirmou o endereço errado.
- **Causa:** configuração copiada de uma VM para a outra sem ajustar o IP.
- **Correção:** acesso temporário com `ip addr add` pelo console, arquivo recriado com `tee` e `netplan apply`; cache ARP do Windows limpo com `netsh interface ip delete arpcache`.
- **Aprendizado:** durante o conflito, o Zabbix chegou a mostrar a vm-agente como UP porque consultava a própria vm-zabbix. Um status UP precisa ser confirmado pelo horário da última coleta.

### 9. IP do host dependente do DHCP
- **Sintoma:** o Windows estava com `192.168.56.102` na Host-Only, recebido pelo DHCP do VirtualBox.
- **Risco:** desligar o DHCP cortaria o acesso do host às VMs.
- **Correção:** adaptador do host fixado em `192.168.56.1` antes de desligar o DHCP.

### 10. vm-wazuh sem rede do lab e sem SSH
- **Sintoma:** a vm-wazuh não respondia em `192.168.56.110`.
- **Causa:** o Adaptador 2 (Host-Only) não estava ativado na VM criada do zero, e o serviço SSH não estava habilitado.
- **Correção:** Adaptador 2 ativado em Host-Only, IP fixo via netplan e `sudo systemctl enable --now ssh`.

### 11. Disco e memória abaixo do planejado
- **Sintoma:** a partição raiz tinha 25 GB de um disco de 50 GB, a VM tinha 4 GB de RAM e nenhum swap.
- **Causa:** o instalador do Ubuntu usa só parte do disco no LVM por padrão.
- **Correção:** partição e sistema de arquivos expandidos (`growpart` e `resize2fs`), RAM aumentada para 6 GB com `VBoxManage modifyvm --memory 6144` (com a VM totalmente desligada), swap de 2 GB com `swappiness` 10 e heap do indexer em 2 GB no `jvm.options`.

### 12. Erro 500 no Dashboard do Wazuh após trocar a senha
- **Sintoma:** depois de trocar a senha do admin com `wazuh-passwords-tool.sh`, o login retornava erro 500.
- **Causa:** o navegador mantinha o cookie da sessão antiga.
- **Correção:** limpeza dos cookies do site (ou janela anônima).

### 13. wazuh-manager em `failed` depois do boot
- **Sintoma:** indexer e dashboard subiam, o manager não.
- **Causa:** no boot, o manager estourava o tempo de inicialização do systemd enquanto o indexer ainda carregava.
- **Correção:** override em `/etc/systemd/system/wazuh-manager.service.d/override.conf`:
  ```ini
  [Unit]
  After=wazuh-indexer.service

  [Service]
  TimeoutStartSec=300
  ```
  seguido de `sudo systemctl daemon-reload`. Validado com reboot: os três serviços ficam ativos.

### 14. Falso positivo de rootcheck na vm-agente
- **Sintoma:** 56 alertas "Trojaned version of file detected" (regra 510, nível 7).
- **Causa:** o Ubuntu 26.04 troca o coreutils pelo rust-coreutils, e o rootcheck procura assinaturas antigas nos binários.
- **Investigação:** `dpkg --verify` sem divergências nos pacotes.
- **Correção:** `<check_trojans>no</check_trojans>` na vm-agente. A integridade dos binários continua coberta pelo FIM. Detalhes no [write-up 03](writeups/03-wazuh-linha-de-base.md).

## Práticas aplicadas

- VM base limpa com snapshot, usada para criar todos os clones
- Clones completos com novos endereços MAC e novo `machine-id`
- Hostname próprio em cada VM
- Snapshots a cada marco validado (`rede-ok`, `fase1-zabbix-grafana-ok`, `fase1-agente-ok`, `fase0-ip-fixo`, `fase4-correlacao`)
- IPs fixos via netplan e DHCP da Host-Only desligado
- MAC único em cada placa de rede dos clones
- Acesso por SSH com chave ed25519 e aliases no `~/.ssh/config` (`wazuh`, `agente`, `zabbix`), sem digitar senha
- VMs ligadas e desligadas sem interface gráfica (`VBoxManage startvm --type headless`)
- Conta de serviço com menor privilégio para integrações
- `.gitignore` e verificação de segredos antes de cada commit
- Regras de detecção versionadas no Git e validadas (`wazuh-analysisd -t`) antes de cada deploy
- Repositório do Wazuh desativado no APT, para evitar atualização sem planejamento

## Status

- [x] VM base com Ubuntu Server e snapshot
- [x] Rede Host-Only e clones (`vm-zabbix`, `vm-agente`)
- [x] SSH funcionando e ping entre as VMs
- [x] Zabbix server na `vm-zabbix` e agente na `vm-agente`
- [x] Grafana com dashboards de CPU, memória, disco, rede, disponibilidade e problemas
- [x] Teste de detecção da queda do agente, com prints
- [x] IPs fixos, DHCP da Host-Only desligado e MACs únicos
- [x] Suricata na vm-agente com regras ET Open e leitura do `eve.json`
- [x] Wazuh all-in-one na vm-wazuh, com o agente lendo os alertas do Suricata
- [x] Regras locais com mapeamento no MITRE ATT&CK (T1033, T1046) e correlação
- [x] Write-ups 02 a 05
- [ ] Ajuste da 100102 com `ignore` e testes de host (força bruta no SSH, FIM em `/etc`, criação de usuário)
- [ ] Regras Sigma
- [ ] Zabbix monitorando a vm-wazuh
- [ ] pfSense com VLANs e IDS/IPS
- [ ] Aplicação web de helpdesk como alvo

## Como reproduzir

1. Instale o VirtualBox e aponte a pasta padrão de máquinas para um disco com espaço livre.
2. Baixe a ISO do Ubuntu Server LTS e crie a VM base (2 GB de RAM, 1 CPU, 20 GB de disco).
3. Instale o sistema, atualize os pacotes, instale o `openssh-server` e tire um snapshot.
4. Ative o Adaptador 2 em Host-Only e crie os clones com novos endereços MAC (confira também o Adaptador 1).
5. Em cada clone, ajuste o hostname e gere um novo `machine-id`.
6. Fixe o IP do host na Host-Only (`192.168.56.1`), configure IP fixo em cada VM via netplan e desligue o DHCP da Host-Only.
7. Teste o SSH a partir do host e o `ping` entre as VMs.
8. Na vm-zabbix, instale Zabbix server, frontend e agente 2 com MariaDB, Apache e PHP-FPM (repositório oficial do Zabbix 7.0). Dê 4 GB de RAM e 2 vCPUs a essa VM.
9. Na vm-agente, instale o Zabbix Agent 2 com `Server` e `ServerActive` apontando para `192.168.56.101` e `Hostname=vm-agente`.
10. Instale o Grafana e o plugin do Zabbix na vm-zabbix, crie o usuário de serviço e o token, e importe o dashboard deste repositório.
11. Na vm-agente, instale o Suricata, ative as regras ET Open com `suricata-update`, ajuste `HOME_NET`, `EXTERNAL_NET` e a interface `enp0s8`.
12. Crie a vm-wazuh (Ubuntu 24.04, 6 GB de RAM, 2 vCPU, 50 GB) e instale o Wazuh all-in-one com o script oficial.
13. Instale o Wazuh Agent na vm-agente apontando para `192.168.56.110` e adicione o `eve.json` ao `ossec.conf`.
14. Copie `wazuh/rules/local_rules.xml` para o Manager, valide e reinicie.

## Evidências

### Painel do Zabbix
![Dashboard do Zabbix](Imagens/zabbix-dashboard.png)

### Hosts monitorados
![Hosts no Zabbix](Imagens/zabbix-hosts.png)

### Dados coletados da vm-agente
![Latest data da vm-agente](Imagens/zabbix-latest-data.png)

## Próximos passos

- **Fase 4 (continuação):** `ignore` na 100102 e detecções de host: força bruta no SSH (T1110), alteração de arquivos em `/etc` e criação de usuário com shell
- **Fase 5:** regras Sigma para registrar as detecções em formato padrão
- **Fase 6:** Zabbix monitorando a vm-wazuh
- **Fase 7:** pfSense com VLANs e IDS/IPS, com um notebook externo como atacante
- **Fase 8:** aplicação web de helpdesk em PHP como alvo
- **Depois:** endpoint Windows com Sysmon e Wazuh Agent, Atomic Red Team, Active Directory e active response do Wazuh

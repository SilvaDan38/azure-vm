![Azure VM](https://img.shields.io/badge/Azure-Virtual%20Machine-0078D4?logo=microsoftazure&logoColor=white)
![Datadog](https://img.shields.io/badge/Datadog-Agent%207-632CA6?logo=datadog&logoColor=white)
![Terraform](https://img.shields.io/badge/Terraform-IaC-844FBA?logo=terraform&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04-E95420?logo=ubuntu&logoColor=white)

# Azure VM Linux + Datadog Agent (cloud-init)

Lab de observabilidade com VM Linux no Azure e Datadog Agent instalado automaticamente via cloud-init no boot.

## Arquitetura

```
┌────────────────────────────────────────────────────────┐
│  Azure (eastus2)                                       │
│                                                        │
│  ┌─────────────────────────────────────────────────┐   │
│  │  VNet 10.0.0.0/16                               │   │
│  │                                                 │   │
│  │  ┌───────────────────────────────────────────┐  │   │
│  │  │  Subnet 10.0.1.0/24                       │  │   │
│  │  │                                           │  │   │
│  │  │  ┌─────────────────────────────────────┐  │  │   │
│  │  │  │  VM Linux (Ubuntu 22.04)            │  │  │   │
│  │  │  │                                     │  │  │   │
│  │  │  │  ┌───────────────────────────────┐  │  │  │   │
│  │  │  │  │  Datadog Agent 7              │  │  │  │   │
│  │  │  │  │  (instalado via cloud-init)   │──┼──┼──┼───┼──▶ Datadog Backend
│  │  │  │  └───────────────────────────────┘  │  │  │   │
│  │  │  └─────────────────────────────────────┘  │  │   │
│  │  └───────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────┘   │
│                                                        │
│  NSG: permite SSH (22) inbound                         │
│  Public IP: Standard SKU (static)                      │
└────────────────────────────────────────────────────────┘
```

## Instrumentação Datadog

O Datadog Agent 7 é instalado automaticamente no primeiro boot da VM via **cloud-init**, sem necessidade de acesso manual à máquina.

### Fluxo de instalação

1. O Terraform renderiza o `cloud-init.yaml` injetando a `DD_API_KEY` como variável no template
2. O cloud-init é passado à VM como `custom_data` (base64-encoded)
3. No boot, o cloud-init executa o script oficial de instalação do Agent 7:

```bash
DD_API_KEY=${dd_api_key} \
DD_SITE="datadoghq.com" \
DD_ENV="lab" \
DD_HOST_TAGS="env:lab,service:vm-lab,version:1.0.0" \
bash -c "$(curl -L https://install.datadoghq.com/scripts/install_script_agent7.sh)"
```

4. O Agent é habilitado e iniciado via systemd (`systemctl enable/start datadog-agent`)

### Configuração aplicada

| Parâmetro | Valor | Descrição |
|-----------|-------|-----------|
| `DD_API_KEY` | Injetada via Terraform variable | Autentica o Agent no backend Datadog |
| `DD_SITE` | `datadoghq.com` | Endpoint do Datadog (US1) |
| `DD_ENV` | `lab` | Tag de ambiente para unified service tagging |
| `DD_HOST_TAGS` | `env:lab,service:vm-lab,version:1.0.0` | Tags aplicadas ao host no Infrastructure Map |

### Como o Agent chega à VM

```
terraform apply
    │
    ▼
templatefile("cloud-init.yaml", { dd_api_key = var.dd_api_key })
    │
    ▼
custom_data = base64encode(rendered_cloud_init)
    │
    ▼
VM boot → cloud-init runcmd → curl install script → Agent rodando
```

### O que o Agent coleta por padrão (sem configuração adicional)

- **System checks**: CPU, memória, disco, rede, load, processos
- **Journald/syslog**: logs do SO e serviços systemd
- **Host metadata**: hostname, OS, cloud provider, tags → visível no Host Map

### Arquivos relevantes na VM (pós-instalação)

| Path | Descrição |
|------|-----------|
| `/etc/datadog-agent/datadog.yaml` | Configuração principal do Agent |
| `/etc/datadog-agent/conf.d/` | Diretório de configurações de integrações |
| `/var/log/datadog/agent.log` | Log do Agent |
| `/opt/datadog-agent/` | Binários do Agent |

## Telemetria Coletada

| Sinal | Fonte | Detalhes |
|-------|-------|----------|
| **Métricas de infra** | Agent (system checks) | CPU, memória, disco, rede, processos |
| **Logs** | Agent (journald/syslog) | Logs do SO e serviços da VM |
| **Host Map** | Agent | Visibilidade do host no Datadog Infrastructure |

## Pré-requisitos

- Azure CLI (`az`) autenticado
- Terraform >= 1.0
- Chave SSH em `~/.ssh/id_rsa.pub`
- Variável de ambiente exportada:
  ```bash
  export DD_API_KEY="<sua-api-key>"
  ```

## Deploy

```bash
cd terraform
terraform init
terraform apply -var="dd_api_key=$DD_API_KEY"
```

## Validação

```bash
# SSH na VM
ssh azureuser@$(terraform output -raw public_ip)

# Verificar status do Agent
sudo systemctl status datadog-agent

# Status detalhado
sudo datadog-agent status

# Verificar conectividade com Datadog
sudo datadog-agent diagnose
```

## Estrutura

```
├── terraform/
│   ├── main.tf           # Infra: RG, VNet, Subnet, NSG, NIC, VM
│   └── cloud-init.yaml   # Instalação do Datadog Agent 7
├── README.md
└── .gitignore
```

## Cleanup

```bash
cd terraform && terraform destroy -var="dd_api_key=$DD_API_KEY"
```

## Segurança

- **Nunca** comitar `DD_API_KEY` — use variáveis de ambiente
- O NSG permite apenas SSH (porta 22) — restrinja `source_address_prefix` ao seu IP em produção
- A chave SSH é lida de `~/.ssh/id_rsa.pub` (não é copiada para o repo)

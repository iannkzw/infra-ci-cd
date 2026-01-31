# Configuração de GitHub Environments e Secrets

Cada ambiente (dev, qa, sbx, prd) deve ter seus próprios **Secrets** e **Variables** no GitHub. O pipeline usa o environment definido pela branch; **ecs_service** = service ECS e task definition; **ecr_repo** = nome do repositório ECR; **ecr_registry** vem da variável do environment.

## Mapeamento branch → environment

| Branch (app) | GitHub Environment | Aprovação |
|--------------|--------------------|-----------|
| `dev` | `dev` | Não |
| `qa` | `qa` | Não |
| `sbx` | `sbx` | Sim (Required reviewers) |
| `prd` | `prd` | Sim (Required reviewers) |

## ecs_service, ecr_repo e ECR

O pipeline recebe **ecs_service** (nome do service ECS e da task definition family), **ecr_repo** (nome do repositório ECR) e **ecr_registry** (URL do ECR). A imagem é enviada para `{ecr_registry}/{ecr_repo}:{tag}`.

- O caller passa `ecs_service` (ex.: `inbound-nfe-api-envioxml`), `ecr_repo` (ex.: `inbound`) e `ecr_registry: ${{ vars.ECR_REGISTRY }}`.
- O repositório ECR deve existir com o nome **ecr_repo** (ou ser criado antes do primeiro push).

## Como configurar

### 1. Criar os 4 environments

Em **Settings > Environments** do repositório da aplicação (ou da organização):

1. Crie os environments: `dev`, `qa`, `sbx`, `prd`.
2. Em **sbx** e **prd**: em **Deployment protection rules**, marque **Required reviewers** e adicione os aprovadores.
3. Em cada environment, em **Deployment branches**, restrinja à branch correspondente (ex.: env `prd` apenas branch `prd`).

### 2. Environment variables (por ambiente)

Em cada environment, em **Environment variables**:

| Variable | Obrigatório | Descrição |
|----------|-------------|-----------|
| `ECR_REGISTRY` | Sim | URL do registry ECR (ex.: `123456789.dkr.ecr.us-east-1.amazonaws.com`) |

### 3. Environment secrets (por ambiente)

Em cada environment, em **Environment secrets**:

| Secret | Obrigatório | Descrição |
|--------|-------------|-----------|
| `AWS_ACCESS_KEY_ID` | Sim | Access Key da conta/role AWS do ambiente |
| `AWS_SECRET_ACCESS_KEY` | Sim | Secret Key correspondente |
| `ECS_CLUSTER` | Não* | Nome do cluster ECS (ex.: `cluster-dev`); pode ser passado como input `ecs_cluster` |
| `ECS_TASK_EXECUTION_ROLE_ARN` | Não* | ARN da role de execução da task; pode ser passado como input `ecs_task_execution_role_arn` |

\* Para o primeiro deploy (service ainda não existe), informe **cluster** via input ou secret. O **ecs_service** é sempre passado como input. A **role de execução** é obrigatória em todo deploy (input ou secret).

### 4. Configuração do serviço ECS (inputs do workflow)

O pipeline suporta criar ou atualizar o service ECS:

- **Service já existe no cluster**: atualiza com nova task definition (imagem do build).
- **Service não existe**: cria a task definition e o service (com rede e, para API, Load Balancer).

**Obrigatório para todo deploy:**

- `ecs_service` (nome do service ECS e da task definition)
- `ecr_repo` (nome do repositório ECR)
- `ecs_task_execution_role_arn` (ou secret `ECS_TASK_EXECUTION_ROLE_ARN`)
- `ecs_cluster` (ou secret `ECS_CLUSTER`)

**Para criar o service (primeiro deploy):**

- `subnet_ids`: IDs das subnets separados por vírgula
- `security_group_ids`: IDs dos security groups separados por vírgula

**Para API (service_type=api) com Load Balancer:**

- `load_balancer_target_group_arn`: ARN do target group do ALB (o path, ex. `/api/nfe/*`, é configurado na listener rule do ALB que aponta para esse target group)
- `container_name`: nome do container na task definition (default: `app`)
- `container_port`: porta exposta (default: `80`)

**Worker (service_type=worker):** não usa Load Balancer; não informe `load_balancer_target_group_arn`.

**Task definition (cenário real ECS):**

- `container_environment`: JSON array de variáveis de ambiente no container (ex.: `[{"name":"INVOISYS_ENV","value":"prd"}]`). Use variáveis do environment ou secret para não expor valores sensíveis no YAML.
- `container_secrets`: JSON array de referências a Secrets Manager (ex.: `[{"name":"ConnectionStrings__PostgreSql","valueFrom":"arn:aws:secretsmanager:..."}]`).
- `runtime_cpu_architecture`, `runtime_os_family`: ex.: X86_64, LINUX (compatível com task definitions Fargate).
- `awslogs_mode`, `awslogs_create_group`, `awslogs_max_buffer_size`: opções de log (ex.: non-blocking, true, 25m).
- `port_mapping_app_protocol`: ex.: http.

**Service (cenário real ECS):**

- `enable_zone_rebalancing`: ativar rebalanceamento de zonas de disponibilidade.
- `health_check_grace_period_seconds`: período de carência do health check (segundos).
- `deployment_circuit_breaker_enable`, `deployment_circuit_breaker_rollback`: disjuntor de implantação e rollback automático.
- `capacity_provider_strategy`: ex.: `FARGATE:0:1,FARGATE_SPOT:0:4` (provider:base:weight). Se informado, o create-service usa capacity provider strategy em vez de apenas launch-type FARGATE; update-service não altera essa configuração.
- `platform_version`: ex.: 1.4.0.
- `enable_execute_command`: ECS Exec (comandos interativos no container).
- `deployment_minimum_healthy_percent`, `deployment_maximum_percent`: ex.: 100, 200.

**Outros inputs opcionais:** `ecs_task_role_arn`, `task_cpu`, `task_memory`, `assign_public_ip`, `desired_count`.

**Logs:** o pipeline usa o log group `/ecs/<nome-do-service>`. Crie o log group no CloudWatch (ou garanta que a role de execução tenha permissão `logs:CreateLogGroup`).

### 5. Repositório de templates (este repo)

- Os **secrets** e **variables** são configurados no **repositório da aplicação** (ou na organização) que chama o workflow.
- O job do caller usa `environment: ${{ github.ref_name }}` e `secrets: inherit`; o caller passa `ecs_service`, `ecr_repo` e `ecr_registry: ${{ vars.ECR_REGISTRY }}`.

### 6. Resumo de segurança

- Zero secrets em plain text nos YAMLs.
- Cada ambiente tem credenciais isoladas.
- sbx e prd exigem aprovação manual antes do deploy e do rollback em prd.

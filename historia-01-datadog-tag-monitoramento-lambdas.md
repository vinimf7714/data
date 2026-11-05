# História 01 — Monitoramento base das Lambdas via tag Datadog

## História

**Título:** Habilitar coleta de métricas do Datadog em todas as Lambdas da aplicação (tag `datadog = enabled`)

**Descrição**

**Como** time responsável pela aplicação,
**quero** que todas as Lambdas tenham a tag de monitoramento do Datadog aplicada via Terraform,
**para** ter visibilidade centralizada da saúde da aplicação (execuções, erros, duração, throttles, concorrência) e poder criar dashboards e alertas básicos sem alterar código.

**Contexto**

A integração AWS do Datadog lê as métricas do CloudWatch usando uma IAM role na nossa conta. A integração está configurada para coletar apenas os recursos que têm a tag de monitoramento. Hoje nossas Lambdas não têm essa tag, por isso não aparecem no Datadog.

Esta história é o primeiro passo da observabilidade da aplicação. A instrumentação de runtime (APM/traces) fica para a História 02.

**Critérios de aceite**

- [ ] Todas as Lambdas da aplicação têm a tag `datadog = enabled` (ou a chave/valor padrão da conta), aplicada via Terraform. Nenhuma foi tagueada manualmente no console.
- [ ] As Lambdas também têm as tags de Unified Service Tagging: `env`, `service` e `version`.
- [ ] A conta AWS está integrada ao Datadog, e o filtro de tag do namespace `AWS/Lambda` inclui a tag usada.
- [ ] As métricas `aws.lambda.invocations`, `aws.lambda.errors`, `aws.lambda.duration` e `aws.lambda.throttles` aparecem no Datadog para todas as funções.
- [ ] Existe um dashboard básico da aplicação com essas métricas.
- [ ] Existem pelo menos dois monitores (alertas): taxa de erro e throttles, roteados para o canal do time.
- [ ] Não houve alteração de código, handler ou layer das Lambdas.

**Fora de escopo**

- APM, traces e enhanced metrics (História 02)
- Métricas customizadas de negócio
- Envio de logs para o Datadog

**Dependências**

- Acesso de leitura ao Datadog
- Confirmar com o time de plataforma/cloud quem é dono da integração AWS ↔ Datadog da conta

---

## Parte técnica

### Precisa de API key?

**Não precisa nas Lambdas nem na conta AWS.** Neste modelo o Datadog é quem busca os dados. Ele assume uma IAM role da nossa conta, validada por External ID, e lê o CloudWatch. Nada é enviado da nossa conta para o Datadog.

| Onde | API key? | APP key? |
|---|---|---|
| Lambdas | ❌ | ❌ |
| IAM role da integração | ❌ (usa External ID) | ❌ |
| Provider Terraform `datadog` (só se **nós** formos configurar a integração ou os monitores via Terraform) | ✅ | ✅ |

> Se a integração for gerenciada por um time central, provavelmente só precisamos taguear as Lambdas. Para criar dashboards e monitores via Terraform, o provider `datadog` precisa de API key + APP key. Pelo console do Datadog não precisa.

### 1. O que verificar na conta AWS

**1.1 A conta já está integrada ao Datadog?**

- No Datadog, vá em **Integrations → Amazon Web Services** e procure o ID da conta.
- Na AWS, procure a role da integração (normalmente `DatadogIntegrationRole` ou o nome padrão da empresa):

```bash
aws iam list-roles --query "Roles[?contains(RoleName, 'atadog')].RoleName"
```

**1.2 Trust policy da role**

A role precisa confiar na conta AWS do Datadog e exigir o External ID:

```json
{
  "Effect": "Allow",
  "Principal": { "AWS": "arn:aws:iam::464622532012:root" },
  "Action": "sts:AssumeRole",
  "Condition": { "StringEquals": { "sts:ExternalId": "<EXTERNAL_ID_GERADO_PELO_DATADOG>" } }
}
```

> `464622532012` é a conta do Datadog para os sites comerciais (US1/US3/US5/EU/AP1). Confirme na documentação do seu site Datadog.

**1.3 Permissões da role (mínimo para Lambda)**

```bash
aws iam list-attached-role-policies --role-name DatadogIntegrationRole
aws iam list-role-policies --role-name DatadogIntegrationRole
```

Confira se existem pelo menos estas ações:

| Ação | Para quê |
|---|---|
| `cloudwatch:GetMetricData`, `cloudwatch:ListMetrics`, `cloudwatch:GetMetricStatistics` | Ler métricas |
| `lambda:List*`, `lambda:GetPolicy` | Descobrir funções e metadados |
| `tag:GetResources`, `tag:GetTagKeys`, `tag:GetTagValues` | **Essencial para o filtro por tag funcionar** |

> Sem `tag:GetResources`, o Datadog não enxerga as tags e o filtro descarta tudo.

**1.4 Filtro de tag da integração**

No Datadog, vá em **Integrations → AWS → (conta) → Metric Collection** e confira:

- Se o namespace `AWS/Lambda` está habilitado.
- Se existe tag filter para Lambda e qual é o formato exato (ex.: `datadog:enabled`).
- Se as regiões das nossas Lambdas estão habilitadas (ex.: `sa-east-1`).

> Anote a chave e o valor **exatos**. A tag na Lambda tem que bater com o filtro, incluindo maiúsculas e minúsculas.

### 2. Passo a passo

**2.1 Centralizar as tags no Terraform**

```hcl
# locals.tf
locals {
  datadog_tags = {
    datadog = "enabled"        # ajustar ao filtro da integração
    env     = var.environment  # dev | hml | prd
    service = var.service_name # nome lógico da aplicação
    version = var.app_version
  }
}
```

**2.2 Aplicar em todas as Lambdas**

```hcl
resource "aws_lambda_function" "processa_arquivo" {
  function_name = "${var.service_name}-processa-arquivo"
  # ... resto igual ...

  tags = merge(var.tags_padrao, local.datadog_tags)
}
```

Se as Lambdas vêm de um módulo interno, adicione a variável no módulo e faça o merge lá dentro. Assim toda Lambda nova já nasce monitorada.

> Evite usar `default_tags` do provider só para isso, porque ele tagueia **todos** os recursos (S3, SQS etc.). Isso pode fazer outros namespaces entrarem na coleta se tiverem filtro parecido e aumentar custo.

**2.3 Inventário: garantir que nenhuma ficou de fora**

```bash
# Lista todas as funções e mostra quais NÃO têm a tag
for fn in $(aws lambda list-functions --query "Functions[].FunctionArn" --output text); do
  tag=$(aws lambda list-tags --resource "$fn" --query "Tags.datadog" --output text)
  [ "$tag" != "enabled" ] && echo "SEM TAG: $fn"
done
```

**2.4 `terraform plan` / `apply`**

- O plan deve mostrar **somente** alteração de `tags` (update in-place) nas Lambdas.
- Se aparecer replace ou alteração de código ou handler, pare e revise.

**2.5 (Opcional, se o time for dono) Integração via Terraform**

```hcl
resource "datadog_integration_aws_account" "conta" {
  aws_account_id = "123456789012"
  aws_partition  = "aws"

  aws_regions { include_only = ["sa-east-1"] }

  auth_config {
    aws_auth_config_role { role_name = "DatadogIntegrationRole" }
  }

  metrics_config {
    namespace_filters { include_only = ["AWS/Lambda"] }
    tag_filters {
      namespace = "AWS/Lambda"
      tags      = ["datadog:enabled"]
    }
  }

  logs_config { lambda_forwarder {} }
  resources_config {}
  traces_config { xray_services {} }
}
```

> O schema desse resource mudou entre versões do provider. Confira na versão que vocês usam.

### 3. Validação

1. Aguarde de 10 a 15 minutos após o apply. A coleta é por polling do CloudWatch.
2. No Datadog, abra **Metrics Explorer** com `aws.lambda.invocations` e agrupe por `functionname`. Todas as funções devem aparecer.
3. Na tela **Serverless**, as funções devem aparecer na lista (só com métricas, sem traces).
4. Filtre por `service:<nome>` e `env:<ambiente>` para garantir que as tags de unified tagging chegaram.

### 4. Dashboard e monitores sugeridos

| Monitor | Query base | Sugestão de limite |
|---|---|---|
| Taxa de erro | `sum:aws.lambda.errors{service:X} / sum:aws.lambda.invocations{service:X}` | > 5% em 10 min |
| Throttles | `sum:aws.lambda.throttles{service:X}` | > 0 em 5 min |
| Duração perto do timeout | `max:aws.lambda.duration.maximum{service:X}` | > 80% do timeout |
| Fila parada (se houver SQS) | `max:aws.sqs.approximate_age_of_oldest_message` | conforme SLA |

### 5. Limitações (motivam a História 02)

- Atraso de alguns minutos, porque a coleta é por polling.
- Mostra **que** deu erro, não **onde** nem **por quê**.
- Não tem cold start, memória real usada, chamadas a dependências (S3, DynamoDB, APIs) nem stack trace.

### 6. Rollback

Remova a tag no Terraform e aplique. O impacto na aplicação é zero, porque só o Datadog para de coletar.

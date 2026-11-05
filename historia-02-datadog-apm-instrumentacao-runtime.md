# História 02 — Instrumentação de runtime das Lambdas (APM / Traces)

## História

**Título:** Instrumentar as Lambdas com Datadog Extension + APM para rastreamento ponta a ponta

**Descrição**

**Como** time responsável pela aplicação,
**quero** instrumentar o runtime das Lambdas com a Datadog Lambda Extension e a biblioteca de tracing,
**para** identificar rapidamente **onde** e **por que** uma execução falhou ou ficou lenta (qual chamada a S3, DynamoDB, SQS ou API externa), acompanhar cold starts e uso real de memória, e correlacionar traces com logs.

**Contexto**

É a evolução da História 01. As métricas do CloudWatch já dizem **que** algo deu errado, mas não dizem onde. A instrumentação adiciona traces por execução, métricas de trace (latência p50/p95/p99, hits e erros por operação) e enhanced metrics da Lambda em tempo quase real.

Ao contrário da História 01, aqui as Lambdas **enviam** dados diretamente ao Datadog. Por isso entram API key, saída de rede e mudança no handler.

**Critérios de aceite**

- [ ] Todas as Lambdas da aplicação usam as layers Datadog Extension e Datadog lib (runtime correspondente), via Terraform.
- [ ] A API key do Datadog está no Secrets Manager. Nenhuma key aparece em texto puro em variável de ambiente, tfvars ou repositório.
- [ ] As Lambdas conseguem alcançar o intake do Datadog (NAT, proxy ou PrivateLink) e isso foi validado com o time de rede.
- [ ] Os traces aparecem em **APM → Traces** com `service`, `env` e `version` corretos.
- [ ] O Service Map mostra as dependências da aplicação (S3, DynamoDB, SQS, HTTP etc.).
- [ ] As enhanced metrics (`aws.lambda.enhanced.*`) aparecem, incluindo cold start e memória usada.
- [ ] A captura de payload está **desligada** (`DD_CAPTURE_LAMBDA_PAYLOAD=false`). Nenhum dado sensível vai para o Datadog.
- [ ] O overhead de cold start e duração foi medido em DEV/HML antes de PRD e registrado na história.
- [ ] A estratégia de logs foi definida para evitar duplicidade, caso já exista Forwarder.
- [ ] O rollout foi feito por ambiente (DEV → HML → PRD), com rollback documentado.

**Fora de escopo**

- Métricas customizadas de negócio (pode ser uma história 03)
- Instrumentação de jobs Spark/Glue (produto diferente: Data Jobs Monitoring)

**Dependências**

- História 01 concluída
- API key do Datadog provisionada pelo time dono da conta Datadog
- Time de rede: liberação de saída para o Datadog
- Time de segurança/FinOps: aceite de envio de traces e custo de APM serverless

---

## Parte técnica

### Precisa de API key?

**Sim.** A extension roda dentro da Lambda e **envia** traces, métricas e (opcionalmente) logs direto para o Datadog, autenticando com a API key.

| Onde | API key? | APP key? |
|---|---|---|
| Lambdas (extension) | ✅ via Secrets Manager | ❌ |
| Role de execução da Lambda | Precisa **ler** o secret | ❌ |
| IAM role da integração (História 01) | ❌ | ❌ |

> A APP key nunca vai para a Lambda. Ela só serve para chamar a API de gerenciamento do Datadog, como o provider Terraform.

### 1. O que verificar na conta AWS

**1.1 Secret da API key**

```bash
aws secretsmanager describe-secret --secret-id datadog/api-key
```

- O valor deve ser a API key em texto puro, não um JSON.
- Anote o ARN e o KMS key usado. Se for CMK, a role da Lambda precisa de `kms:Decrypt`.
- Se o secret estiver em outra conta (comum em ambientes centralizados), a resource policy do secret e a key policy do KMS precisam liberar a role da Lambda.

**1.2 Role de execução das Lambdas**

Adicione esta policy:

```json
{
  "Effect": "Allow",
  "Action": "secretsmanager:GetSecretValue",
  "Resource": "arn:aws:secretsmanager:sa-east-1:<conta>:secret:datadog/api-key-*"
},
{
  "Effect": "Allow",
  "Action": "kms:Decrypt",
  "Resource": "<arn-da-cmk-se-houver>"
}
```

**1.3 Rede: ponto mais comum de falha em ambiente bancário**

Para cada Lambda, verifique se ela está em VPC:

```bash
aws lambda get-function-configuration --function-name <fn> --query "VpcConfig"
```

| Situação | O que precisa |
|---|---|
| Fora de VPC | Funciona direto (internet da AWS) |
| Em VPC com NAT | Liberar egress HTTPS (443) para os domínios do site Datadog |
| Em VPC sem internet | PrivateLink do Datadog ou proxy corporativo (`DD_PROXY_HTTPS`) |
| Precisa ler o secret | VPC endpoint do Secrets Manager ou rota até ele |

> Se a extension não alcançar o Datadog, a Lambda **não quebra**, mas os dados se perdem e a extension pode atrasar o fim da execução enquanto tenta fazer flush. Valide em DEV.

**1.4 Runtime e arquitetura**

```bash
aws lambda list-functions --query "Functions[].[FunctionName,Runtime,Architectures[0]]" --output table
```

Isso define quais layers usar (x86_64 vs arm64, Python 3.x vs Node etc.).

**1.5 X-Ray**

Se as Lambdas já usam X-Ray, decida entre manter os dois (com `DD_MERGE_XRAY_TRACES=true`) ou migrar. Não deixe isso implícito.

### 2. Passo a passo

**2.1 Variáveis comuns**

```hcl
locals {
  dd_region_account = "464622532012" # conta que publica as layers Datadog (comercial)

  dd_layers_python = [
    "arn:aws:lambda:${var.region}:${local.dd_region_account}:layer:Datadog-Extension:${var.dd_extension_version}",
    "arn:aws:lambda:${var.region}:${local.dd_region_account}:layer:Datadog-Python312:${var.dd_python_layer_version}",
  ]
  # arm64: Datadog-Extension-ARM / Datadog-Python312-ARM
  # Node:  Datadog-Node20-x / Datadog-Node22-x

  dd_env = {
    DD_API_KEY_SECRET_ARN      = var.dd_api_key_secret_arn
    DD_SITE                    = var.dd_site        # ex.: datadoghq.com
    DD_ENV                     = var.environment
    DD_SERVICE                 = var.service_name
    DD_VERSION                 = var.app_version
    DD_TRACE_ENABLED           = "true"
    DD_ENHANCED_METRICS        = "true"
    DD_CAPTURE_LAMBDA_PAYLOAD  = "false"            # NÃO enviar payload (dados sensíveis)
    DD_LOGS_INJECTION          = "true"             # trace_id nos logs → correlação
    DD_SERVERLESS_LOGS_ENABLED = var.dd_logs_via_extension # ver item 2.4
    DD_MERGE_XRAY_TRACES       = "false"
  }
}
```

> As versões das layers mudam com frequência. Pegue as atuais na documentação "Instrumenting Python/Node Serverless Applications" do Datadog e **fixe a versão**. Não use "latest".

**2.2 Aplicar na Lambda (Python): troca do handler**

A lib Datadog "embrulha" o handler original:

```hcl
resource "aws_lambda_function" "processa_arquivo" {
  function_name = "${var.service_name}-processa-arquivo"
  runtime       = "python3.12"

  handler = "datadog_lambda.handler.handler"   # handler do Datadog
  layers  = concat(local.dd_layers_python, var.outras_layers)

  environment {
    variables = merge(
      var.env_da_aplicacao,
      local.dd_env,
      { DD_LAMBDA_HANDLER = "app.main.lambda_handler" } # handler ORIGINAL
    )
  }

  tags = merge(var.tags_padrao, local.datadog_tags) # da História 01
}
```

Para Node, o handler passa a ser `/opt/nodejs/node_modules/datadog-lambda-js/handler.handler`, mantendo `DD_LAMBDA_HANDLER` com o original.

> ⚠️ **Risco principal:** se o handler for trocado e a layer não estiver presente (ou for de runtime ou arquitetura errada), a função **para de funcionar** (`Runtime.ImportModuleError`). Handler e layers precisam mudar **juntos** no mesmo apply.

**Alternativa:** existe um módulo Terraform oficial do Datadog (`DataDog/lambda-datadog/aws`) que encapsula `aws_lambda_function` e faz layers, handler e variáveis automaticamente. Avalie se ele é compatível com o módulo interno de Lambda de vocês.

**2.3 Policy na role de execução**

```hcl
resource "aws_iam_role_policy" "dd_api_key" {
  role = aws_iam_role.lambda_exec.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      { Effect = "Allow", Action = "secretsmanager:GetSecretValue", Resource = var.dd_api_key_secret_arn },
      # { Effect = "Allow", Action = "kms:Decrypt", Resource = var.dd_secret_kms_arn },
    ]
  })
}
```

**2.4 Decidir a estratégia de logs**

| Cenário | Configuração |
|---|---|
| Já existe Datadog Forwarder lendo os CloudWatch Logs | `DD_SERVERLESS_LOGS_ENABLED=false` (evita log duplicado e custo dobrado) |
| Não existe envio de logs | `true`, se aprovado por segurança (revisar se há dado sensível nos logs) |
| Logs não podem ir ao Datadog | `false` |

**2.5 Rollout**

1. **DEV:** aplicar em 1 Lambda e validar (seção 3).
2. **DEV:** medir overhead (seção 4) e aplicar no restante.
3. **HML:** aplicar em todas e rodar o fluxo ponta a ponta.
4. **PRD:** aplicar em janela combinada, monitorando erro e duração por 24h.

### 3. Validação

1. Invoque a função:
   ```bash
   aws lambda invoke --function-name <fn> --payload '{}' /tmp/out.json
   ```
2. Nos logs do CloudWatch, procure linhas `DATADOG` / `DD_EXTENSION`. Erros de API key ou de conexão aparecem aqui.
3. Em **APM → Traces**, filtre por `service:<nome> env:<ambiente>` e abra um trace: os spans de S3, DynamoDB e HTTP devem aparecer.
4. Em **APM → Service Map**, as dependências devem aparecer.
5. Em **Metrics**, confira `aws.lambda.enhanced.invocations`, `aws.lambda.enhanced.init_duration` (cold start) e `aws.lambda.enhanced.max_memory_used`.
6. Em **Logs** (se habilitado), clique em um log: ele deve abrir o trace correspondente via `dd.trace_id`.

**Troubleshooting rápido**

| Sintoma | Causa provável |
|---|---|
| `Runtime.ImportModuleError` | Layer ausente ou errada; handler trocado sem layer |
| Log `403 Forbidden` / `API key invalid` | `DD_SITE` errado ou key errada |
| Timeout ao final da execução / nada no Datadog | Sem rota de rede para o intake |
| `AccessDenied` no secret | Falta `GetSecretValue` / `kms:Decrypt` ou resource policy cross-account |
| Traces sem service correto | `DD_SERVICE`/`DD_ENV` ausentes |

### 4. Overhead e custo (registrar na história)

- **Cold start:** a extension e a lib adicionam tempo de init. Meça `init_duration` antes e depois.
- **Memória:** a extension consome alguns MB. Em funções com pouca memória configurada, reavalie.
- **Duração:** o flush pode adicionar milissegundos. Compare p95 antes e depois.
- **Custo Datadog:** APM serverless é cobrado à parte da infraestrutura. Valide com FinOps / dono do contrato antes de PRD.

### 5. Rollback

1. Restaurar o handler original, remover `DD_LAMBDA_HANDLER` e as layers Datadog **no mesmo apply**.
2. Remover as variáveis `DD_*` (opcional; sem a layer elas são ignoradas).
3. As métricas da História 01 continuam funcionando, porque não dependem desta história.

### 6. Próximos passos sugeridos

- Métricas de negócio com a lib (`lambda_metric("registros.processados", n, tags=[...])`) — história 03.
- Monitores de APM: latência p95 por operação e taxa de erro por endpoint/recurso.
- SLOs da aplicação usando trace metrics.

# data


# Proposta — democratização dos dados do bloco M

## Objetivo

Levar os resultados calculados do bloco M para o bucket mensalmente, com rastreabilidade de qual versão de de-para gerou cada resultado.

O relacionamento que precisa existir:

```
de-para (versão escolhida pelo cliente)
  → execução (usuário, data/hora, parâmetros)
    → resultado dos blocos M
```

## Pendências conhecidas

- Os lambdas do bloco M ainda operam com dados fake.

Isso não bloqueia a proposta. O contrato de metadado e o fluxo de materialização podem ser construídos e testados com os dados atuais, e passam a valer sem alteração quando os dados reais entrarem. O que estamos definindo aqui é o formato do resultado e sua rastreabilidade, não o cálculo.

---

## Decisão central: versionar o artefato, não o estado quente

O TXT da ECF é a materialização completa do resultado — todos os blocos, todos os registros, no formato que a Receita valida. Se guardamos o TXT, o estado de saída está capturado por inteiro. O Dynamo é memória de trabalho.

**Consequência para a fase 1: não versionamos o Dynamo.** Versionamos o TXT e o Parquet derivado dele, e amarramos tudo por uma tabela de metadado.

O que isso entrega:
- **Auditabilidade completa** — o que foi gerado, quando, por quem, com qual de-para.
- **Diff entre versões** — com dois TXT guardados, comparar registro a registro entrega "o que mudou da v1 para a v2" sem esforço adicional. Negócio vai pedir isso.

O que isso não entrega:
- **Reprodutibilidade** — reprocessar não garante o mesmo resultado, porque o Dynamo mudou por baixo. Aceitável na fase 1, desde que seja decisão consciente e não descoberta.

Isso substitui a alternativa de repartir a tabela quente do Dynamo por versão. Aquele caminho exige backfill de dados legados em sistema com prazo legal em 31 de julho — risco desproporcional para o ganho nesta fase.

---

## Escopo do versionamento

O de-para vale a partir do bloco M, mas o que precisa ser versionado é o **fecho transitivo**:

| Bloco | Situação |
|---|---|
| M | depende direto do de-para |
| N | inteiramente derivado de M |
| X / Y | apenas os registros que consomem apuração |

O `Y730`, novo no leiaute 12, lista os donatários das deduções usadas no cálculo de IRPJ/CSLL — que vêm do N, que vem do M. Mudou o de-para, o `Y730` muda junto. Não podemos tratar o escopo como "só o M".

**Blocos 0 a L não são versionados, mas precisam de fingerprint.** Um hash do estado da base guardado no metadado da execução. Se alguém recuperar uma ECD nova e o `L300` mudar, todas as execuções de M viram lixo silenciosamente. Com o hash, a execução é marcada como `STALE` e a interface avisa antes de gerar o TXT.

---

## Modelo de metadado

Hoje a informação está no S3. Vai para uma tabela do Dynamo:

```
PK: ECF#<cnpj>#AC#<ano_calendario>
SK: EXEC#<ulid>

  competencia            ano-calendário da obrigação (≠ data de processamento)
  dt_processamento
  status                 PROCESSANDO | COMMITADA | INCONSISTENTE | FALHA | STALE
  origem                 sistema | <matrícula>
  disparado_por
  tipo_ecf
  ind_dec                0 original | 1 retificadora
  versao_depara          { plano_fiscal: "341_v3", contas: "341_v2" }
  fingerprint_base       hash dos blocos 0 a L
  bloco_zero             snapshot inline
  s3_txt                 s3://.../exec=<ulid>/ecf.txt
  hash_txt               sha256
  s3_sot                 s3://.../spec/ecf/.../versao=<ulid>/
  qtd_registros_bloco
  inconsistencias
  status_transmissao
  recibo_rf

SK: PONTEIRO#VIGENTE     execucao_id + histórico de promoções
```

Notas sobre campos que não estavam na versão anterior:

- **`competencia` separada de `dt_processamento`.** A ECF do AC 2025 pode ser processada dez vezes ao longo de 2026. Sem a competência não conseguimos listar "todas as versões da ECF 2025", e duas execuções no mesmo dia colidem.
- **`versao_depara` como mapa, não escalar.** A empresa pode subir mais de um de-para (plano fiscal, contas).
- **ULID como identificador**, não contador `v1`/`v2`. Ordenável, carrega tempo, não exige coordenação.
- **`hash_txt`.** Sem ele não provamos que o arquivo não foi alterado, e não deduplicamos reprocessamento idêntico.
- **`bloco_zero` inline.** É o único input que muda o significado de tudo abaixo — o `0010` define o regime. É pequeno o suficiente para caber no item, e recupera parte da linhagem que abrimos mão ao não versionar o Dynamo.
- **`status_transmissao` + `recibo_rf`.** Baratos agora, caros de retroagir. Depois que o TXT foi para a Receita, aquela execução congela: não pode ser reprocessada no lugar.

---

## Lambda de trigger (último do fluxo do bloco M)

**1. Validar a última versão do cálculo consolidado**

Verificar `fingerprint_base` contra o estado atual dos blocos 0 a L. Se não bate, marcar `STALE` e abortar com mensagem clara.

**2. Recuperar a versão de de-para escolhida**

Por competência, empresa e versão. O double check citado no item 2.x deixa de ser necessário para a parametrização do bloco zero — ela vai gravada inline no metadado da execução, então é leitura, não reconciliação.

**3. Gerar o TXT e derivar o restante**

Este é o ponto onde a proposta original tinha "parquet? discutir". A decisão:

**Gerar o TXT primeiro e derivar o Parquet do TXT**, em vez de escrever os dois em paralelo a partir do Dynamo.

O TXT da ECF é delimitado por pipe com o tipo de registro no primeiro campo — parsear é determinístico e barato. A propriedade que ganhamos: o SOT nunca diverge do que foi transmitido, porque veio dele. Se a camada analítica tiver bug, re-derivamos sem tocar no artefato fiscal.

Trade-off: um passo de parse a mais. Se o TXT for grande — plano de contas de banco costuma ser — a derivação não cabe em Lambda e vira task Fargate com parse em streaming. Ainda mais barato que Glue.

**4. Commit**

Escrever os artefatos, e só depois atualizar o ponteiro com conditional write. Consumidor nunca enxerga versão pela metade.

### Sobre reaproveitar o lambda de conversão existente

Não empilhar dois gatilhos no mesmo handler. O lambda atual tem contrato próprio (arquivo chega na SOR → converte → SOT), com SLA e modo de falha diferentes.

Extrair o writer Parquet para um **Lambda Layer compartilhado** e manter dois handlers finos. Código compartilhado, ciclo de vida separado.

---

## Respostas aos pontos 3.1 a 3.4 da versão anterior

**3.2 — os lambdas fazem upsert por código da empresa e data de fechamento. Manter?**

Sim, na fase 1. Lambda é at-least-once, então `put_item` com chave natural continua sendo o idioma certo para idempotência em retry.

**3.3 — teremos use case que precisa das versões anteriores no Dynamo?**

Não na fase 1. Esta é a pergunta que a decisão central responde: as versões anteriores vivem no S3 como TXT e Parquet, não no Dynamo. O Dynamo guarda só o estado corrente.

**3.4 — precisamos ajustar os lambdas para não deletar dados antigos?**

Não agora. Como consequência direta de 3.3, os lambdas continuam como estão. Este ajuste só entra na fase 2, junto com `execucao_id` na chave.

---

## Bloqueador — resolver antes de tudo

O de-para está no S3 versionado por `<codigo_empresa>_v1`, `_v2`, vindo de upload Excel.

Neste desenho, a referência ao de-para é a **única** linhagem de input que guardamos. Se alguém puder sobrescrever o objeto `341_v2`, o metadado passa a apontar para uma mentira e a auditoria inteira cai.

**Regra: de-para é imutável por versão.** Nova edição gera `v3`, nunca reescreve `v2`. Se o portal hoje permite substituir, este é o primeiro item da lista — antes do metadado, antes da materialização.

---

## Complementos de armazenamento

- **S3 Object Versioning**: ligar como rede de segurança contra delete acidental. Não é o mecanismo de versionamento — é por objeto, sem atomicidade de conjunto, e o Athena sempre lê o objeto corrente.
- **S3 Object Lock em modo Compliance** no TXT transmitido e no recibo da Receita. WORM real, é o que se mostra em fiscalização. No Parquet derivado não é necessário — ele é reconstruível.
- **Partition projection** no Glue Catalog, particionando por empresa, competência e versão. Elimina crawler e chamadas de API de partição, o que responde à preocupação de custo do Glue.
- **Lifecycle**: vigente e último ano em Standard, resto em Glacier Instant Retrieval. Deep Archive tem recuperação em horas — inaceitável para consulta de versão pelo cliente.

---

## Fase 2 — fora do escopo agora

Quando o cliente puder voltar versões, a operação **não** é mover o ponteiro para trás.

Depois de transmitida, "voltar para a versão de ontem" é juridicamente uma **retificadora**: ECF nova, com `IND_DEC` marcado e o número do recibo da original. O sistema anda para frente mesmo quando o usuário pensa "voltar".

Então a fase 2 é: **clonar para frente**. Ler o resultado guardado de uma execução antiga e reescrevê-lo como execução nova. O que se lê é exatamente o TXT e o Parquet que a fase 1 já produz.

O que falta construir na fase 2:
- `execucao_id` na PK do Dynamo, com backfill do legado
- Job de rehidratação (Parquet → Dynamo sob novo `execucao_id`)
- Grafo entre exercícios: qual execução do ano N originou os saldos de abertura do ano N+1 (`M500` → `E020`), com invalidação em cascata
- Fluxo de retificadora e segregação de função para quem promove

Nada disso é retrabalho. A fase 1 constrói a primeira metade do rollback.

---

## Ordem de implementação

A ordem não é a ordem do desenho:

1. **Travar imutabilidade do de-para** no portal — tudo abaixo depende disso ser verdade
2. **Tabela de metadado** — barata, já entrega histórico e auditoria
3. **Materialização do TXT** com hash e commit atômico
4. **Derivação Parquet → SOT** — pode vir por último, já que o TXT sozinho garante a auditoria

## Pontos que ainda precisam de decisão

- Volumetria do TXT por empresa, para decidir entre Lambda e Fargate no passo de derivação
- Política de retenção do Parquet no SOT frente à retenção fiscal
- Quem tem permissão de disparar processamento, e se isso já precisa ser diferente de quem transmite

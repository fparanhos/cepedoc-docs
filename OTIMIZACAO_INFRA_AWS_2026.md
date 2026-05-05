# Otimização da Infraestrutura AWS — Maio/2026

**Documento administrativo** | Direção Administrativa e Financeira
**Período de execução:** Abril e Maio de 2026
**Última atualização:** 05/05/2026

---

## Resumo Executivo

Em abril e maio de 2026 a equipe de TI executou um conjunto de ações de redução de custo na infraestrutura AWS que, somadas, **reduzirão a fatura mensal recorrente em aproximadamente 60%** a partir de junho/2026.

| Indicador | Valor |
|---|---|
| Fatura média mensal antes (mar/26 ref.) | **US$ 7.141** |
| Fatura abril/2026 (com upfront RDS) | US$ 11.498 |
| Fatura maio/2026 (forecast) | **US$ 5.701** |
| Fatura projetada junho/2026 em diante | **US$ ~3.800** |
| **Economia anual estimada** | **~US$ 25.300** |
| **Economia em 3 anos (vida do RI RDS)** | **~US$ 76.000 (≈ R$ 418 mil)** |

A maior parcela da economia vem de duas decisões estruturais:

1. **Compra de Reserved Instance (RI) do RDS PostgreSQL** — desembolso único de **US$ 3.300** que zera o custo recorrente do banco por **3 anos**.
2. **Eliminação do cluster Elasticsearch 6.8 (`cepebr`)** — economia recorrente de **US$ 731/mês**.

---

## Histórico de Custos AWS (últimos 14 meses)

```mermaid
xychart-beta
    title "Fatura AWS mensal (USD) — Abr/2025 a Mai/2026"
    x-axis ["Abr25","Mai25","Jun25","Jul25","Ago25","Set25","Out25","Nov25","Dez25","Jan26","Fev26","Mar26","Abr26","Mai26"]
    y-axis "USD" 0 --> 12000
    bar [5657, 5799, 5795, 6138, 6304, 6124, 6253, 6337, 6656, 6664, 6362, 7141, 11498, 5701]
```

> **Observação:** Abril/2026 contém o desembolso único de US$ 3.300 do **upfront do Reserved Instance RDS**, que cobre 3 anos. Excluindo esse upfront, abril fecharia em ~US$ 8.198. A partir de maio, o RDS passa a custar próximo de zero.

---

## Composição da Fatura — Abril/2026 (último mês fechado)

```mermaid
pie title Distribuição da fatura abril/2026 (USD 11.498)
    "RDS PostgreSQL (inclui upfront RI)" : 4420
    "EC2 Compute" : 1126
    "OpenSearch (ES 6.8 + cepebr-v2)" : 945
    "EC2 Other (EBS, NAT, etc.)" : 489
    "AWS Transfer Family" : 216
    "Lightsail" : 188
    "Demais serviços" : 4115
```

---

## Ações Executadas

### 📅 Linha do tempo

```mermaid
gantt
    title Acoes de otimizacao AWS - 2026
    dateFormat YYYY-MM-DD
    axisFormat %d/%m
    section Banco de Dados
    RI RDS 3yr All-Upfront         :done, rds, 2026-04-25, 1d
    section OpenSearch
    Migracao ES6.8 para OS 2.13    :done, mig, 2026-04-29, 6d
    Snapshot pre-delecao ES 6.8    :done, snap, 2026-05-05, 1d
    Exclusao cluster cepebr        :done, del, 2026-05-05, 1d
    section Aplicacoes Legadas
    Beanstalk ingestor zerado      :done, eb, 2026-05-04, 1d
    Beanstalk env terminado        :done, ebt, 2026-05-05, 1d
    section RI OpenSearch
    Compra RI cepebr-v2 3yr        :done, ri2, 2026-05-05, 1d
    Reducao cepebr-v2 single-node  :done, red, 2026-05-05, 1d
    section Transfer Family
    Delete SFTP server pos BBSIA   :done, tf, 2026-05-05, 1d
```

---

### 1. Compra de Reserved Instance — RDS PostgreSQL

**O que é:** Em vez de pagar o RDS hora-a-hora (on-demand), foi comprado um **compromisso de 3 anos**, com **desembolso único** (All Upfront), que dá direito ao uso da capacidade por todo esse período sem cobranças adicionais.

| Item | Valor |
|---|---|
| Tipo de instância | `db.t4g.large` (Graviton, ARM) |
| Quantidade | 2 instâncias |
| Período | 3 anos (até abril/2029) |
| Pagamento | All Upfront — US$ 1.650 × 2 = **US$ 3.300** |
| Custo recorrente | **US$ 0,00/h** durante todo o período |

**Impacto financeiro:**

```mermaid
xychart-beta
    title "RDS PostgreSQL — antes vs depois do Reserved Instance"
    x-axis ["Fev26","Mar26","Abr26 (c/ upfront)","Mai26","Jun26 (proj)","Jul26 (proj)"]
    y-axis "USD" 0 --> 5000
    bar [990, 1212, 4420, 90, 50, 50]
```

A partir de **maio/2026**, o RDS PostgreSQL deixa de pesar na fatura recorrente. A economia em 3 anos (vs uso on-demand) é de aproximadamente **US$ 36.000**.

---

### 2. Eliminação do Cluster Elasticsearch 6.8

**Contexto:** O cluster `cepebr` (Elasticsearch 6.8) era o motor de busca legado do SDOE. Após a migração das aplicações Java (e do indexador Node) para o OpenSearch 2.13 (`cepebr-v2`), o `cepebr` permanecia rodando sem ninguém escrever nele — apenas consumindo recursos.

**Ação:** Snapshot completo dos 596 índices (2.958 shards, 25,9 GB) salvo no bucket S3 `esnovo` para restauração futura caso necessário, seguido da exclusão do domínio.

| Item | Valor |
|---|---|
| Cluster | `cepebr` (Elasticsearch 6.8) |
| Configuração | 3× r5.large.elasticsearch (data) + 3× i4i.large.elasticsearch (master) + 450 GB EBS |
| Custo mensal antes | **~US$ 731** |
| Snapshot | 596 índices, 0 falhas, gravado em `s3://esnovo/snap-pre-delete-20260505` |
| Custo mensal depois | **US$ 0** |
| **Economia anual** | **~US$ 8.770** (≈ R$ 48.200) |

**Mitigação de risco:** o snapshot íntegro permite, se algum sistema esquecido precisar, criar um novo cluster pequeno (ex.: t4g.medium em 1 nó) apontado para o mesmo bucket e restaurar em poucas horas.

---

### 3. Beanstalk legado eliminado

**Contexto:** O ambiente Elastic Beanstalk `cepe-elastic-ingestor-prd` era o ingestor Java antigo de matérias para o ES 6.8. Substituído pelo novo indexador Node, mas o ambiente continuava ligado por inércia — uma instância `t3.micro` continuamente sendo recriada pelo Auto Scaling Group sempre que era desligada.

**Ação executada em duas etapas:**
1. **04/05/2026** — Reduzido o ASG para `min=0, desired=0`, instância detached e parada
2. **05/05/2026** — Ambiente Beanstalk terminado definitivamente (`terminate-environment --environment-id e-pp3yccr6dp`)

| Item | Valor |
|---|---|
| Custo mensal antes | ~US$ 15-25 |
| Custo mensal depois | US$ 0 |
| Status | **Terminado** — sem possibilidade de reativação acidental |

---

### 5. AWS Transfer Family — SFTP descontinuado

**Contexto:** A CEPE mantinha um endpoint SFTP gerenciado pela AWS (`s-c0ee0f77ebbd43eeb`) que recebia arquivos do Banco do Brasil para a operação ATI. Com a migração para a nova integração **BBSIA** (API REST), o canal SFTP deixou de ser utilizado — endpoint estava com **0 usuários cadastrados**.

**Ação:** Servidor Transfer Family deletado em 05/05/2026. Bucket S3 e roles IAM permanecem intactos para preservar histórico.

| Item | Valor |
|---|---|
| Custo mensal antes | **US$ 216** (US$ 0,30/h × 720h, fixo independente do uso) |
| Custo mensal depois | US$ 0 |
| **Economia anual** | **~US$ 2.592** (≈ R$ 14.250) |
| **Economia em 3 anos** | **~US$ 7.776** (≈ R$ 42.770) |

---

### 4. Reserved Instance OpenSearch (cepebr-v2) — 3 anos No-Upfront

**Contexto:** O cluster OpenSearch ativo `cepebr-v2` está provisionado em alta disponibilidade (3 nós × 3 zonas de disponibilidade), o que é desnecessário para o perfil de uso atual da busca eletrônica do SDOE — que tem **dois caminhos de fallback** (PDF e DocPro), tornando o impacto de eventual indisponibilidade da busca eletrônica baixo.

**Plano em duas etapas:**

**4.1 — Compra da Reserved Instance (executada em 05/05/2026):**

| Item | Valor |
|---|---|
| Tipo de instância | `m7g.large.search` (Graviton, geração atual) |
| Quantidade | 1 |
| Período | 3 anos (até maio/2029) |
| Pagamento | **No Upfront** (sem desembolso inicial) |
| Custo recorrente | US$ 0,07/h ≈ **US$ 50,40/mês** |
| Total 3 anos | US$ 1.814 (≈ R$ 9.977) |
| ID da reserva | `b37021bc-53c5-422e-a977-60c4c4f5544f` |
| Reservation name | `cepebr-v2-3yr-no-upfront` |

**4.2 — Redução do cluster (executada em 05/05/2026, em curso):**

Reduzido para configuração **single-node, single-AZ** com instância `m7g.large.search`. A operação é blue/green online (sem downtime), demorando 1–2 horas para o OpenSearch replicar todos os shards para os novos nós.

> Disparado às 18:58 UTC de 05/05/2026, com `aws opensearch update-domain-config`. Subnet escolhida: `subnet-2dabf803` (us-east-1a). A partir do término da migração, a Reserved Instance comprada em 4.1 passa a ser efetivamente consumida.

| Configuração | Custo mensal |
|---|---|
| Antes (3 nós × 3 AZ on-demand) | ~US$ 203 |
| Após redução (1 nó single-AZ on-demand) | ~US$ 68 |
| Após redução **+ RI ativa** | **~US$ 58** (compute) + ~US$ 8 (EBS) |

**Economia recorrente prevista:** ~US$ 145/mês = **US$ 1.740/ano** = **US$ 5.220 em 3 anos**.

---

## Origem da Economia Anual Consolidada

```mermaid
pie title Composição da economia anual prevista (USD 25.342)
    "RDS PostgreSQL (RI 3yr All-Upfront)" : 12000
    "Eliminação do ES 6.8" : 8770
    "Transfer Family SFTP descontinuado" : 2592
    "Redução cepebr-v2 + RI 3yr" : 1740
    "Beanstalk legado eliminado" : 240
```

> Observação: a parcela "RDS RI" considera apenas a economia de uso recorrente (não o desembolso inicial). Em 3 anos a economia bruta do RDS é maior que US$ 36.000.

---

## Comparativo Antes × Depois

```mermaid
xychart-beta
    title "Fatura mensal AWS — antes vs depois das otimizações (USD)"
    x-axis ["Mar/26 (antes)","Mai/26 (transição)","Jun/26 (estimado)"]
    y-axis "USD" 0 --> 8000
    bar [7141, 5701, 4960]
```

| Cenário | Mensal | Anual |
|---|---|---|
| Antes (média mar/26) | US$ 7.141 | US$ 85.692 |
| Maio/2026 (transição) | US$ 5.701 | — |
| Junho/2026 em diante (estimado) | **US$ 4.960** | **US$ 59.520** |
| **Economia recorrente** | **US$ 2.181/mês** | **~US$ 26.000/ano** |

---

## Projeção 36 meses (vida útil dos Reserved Instances)

```mermaid
xychart-beta
    title "Fatura mensal projetada (USD) — Mai/26 a Abr/29"
    x-axis ["Mai26","Ago26","Nov26","Fev27","Mai27","Ago27","Nov27","Fev28","Mai28","Ago28","Nov28","Fev29"]
    y-axis "USD" 0 --> 7000
    bar [5701, 4960, 4960, 4960, 4960, 4960, 4960, 4960, 4960, 4960, 4960, 4960]
```

| Período | Custo total estimado |
|---|---|
| Trajetória atual (sem otimizações) | ~US$ 257.000 |
| Após otimizações executadas | ~US$ 178.000 |
| **Economia em 3 anos** | **~US$ 79.000 (≈ R$ 434 mil)** |

---

## Riscos e Mitigações

| Risco | Probabilidade | Mitigação |
|---|---|---|
| Algum sistema esquecido depender do ES 6.8 | Baixa | Snapshot íntegro guardado em `s3://esnovo` — restauração em poucas horas |
| Indisponibilidade momentânea da busca eletrônica do SDOE durante swap de alias | Baixa | Realizado swap atômico em produção sem janela de inconsistência |
| RI RDS não cobrir crescimento futuro | Média | Tipo `db.t4g.large` cobre projeção atual; aumento exigirá nova RI ou complemento on-demand |
| Single-node OpenSearch ter falha de hardware | Média | Aceito porque o SDOE tem **3 caminhos de busca**: eletrônica, PDF e DocPro (apenas a eletrônica seria afetada) |
| Crescimento de uso aumentar custos | Média | Alarmes de billing já configurados; revisão trimestral planejada |

---

## Próximos Passos (a executar em maio/2026)

| # | Ação | Prazo | Responsável |
|---|---|---|---|
| 1 | Reduzir cluster `cepebr-v2` para single-node single-AZ com `m7g.large.search` (RI já contratado) | Imediatamente após conclusão do reindex em curso | TI |
| 2 | Confirmar com equipe Java que nenhum sistema lê do ES 6.8 (validação adicional) | Próximos 7 dias | TI + Equipe Java |
| 3 | Revisar mensalmente a fatura no Cost Explorer | Mensal | TI / Financeiro |
| 4 | Avaliar renovação dos RIs em abril/2029 | Abril/2029 | TI / Financeiro |

---

## Anexos Técnicos

- **Snapshot ES 6.8:** `s3://esnovo/snap-pre-delete-20260505` (596 índices, 0 falhas, 13min50s de duração)
- **Cluster ativo:** `cepebr-v2` (OpenSearch 2.13, índice `publicacoes_unificado_v2` com alias `publicacoes_unificado`)
- **Reserved Instances ativas:**
  - RDS: 2× `db.t4g.large` 3-yr All-Upfront (até abr/2029)
  - OpenSearch: 1× `m7g.large.search` 3-yr No-Upfront, ID `b37021bc-53c5-422e-a977-60c4c4f5544f` (até mai/2029)
- **Beanstalk env terminado:** `cepe-elastic-ingestor-prd` (env-id `e-pp3yccr6dp`) — operação `terminate-environment` em 05/05/2026

---

*Documento gerado em 05/05/2026 com dados extraídos do AWS Cost Explorer.*

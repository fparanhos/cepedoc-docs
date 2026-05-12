# Projeto: Sistema de Inteligência para Diários Oficiais de Pernambuco

**Versão:** 1.0  
**Data:** Maio de 2026  
**Período coberto:** 2019 – presente  
**Stack base:** OpenSearch 2.xx + Claude (Anthropic) + Python

---

## 1. Visão geral

O projeto transforma o acervo de Diários Oficiais do Estado de Pernambuco — já estruturado matéria por matéria no OpenSearch — em um sistema de inteligência capaz de responder perguntas complexas em linguagem natural sobre um período de quase 7 anos de publicações.

### Problemas que resolve

| Problema | Solução |
|---|---|
| Qual decreto está vigente hoje sobre tema X? | Grafo de vigência com campo `vigente: bool` atualizado a cada publicação |
| Qual decreto revogou/substituiu o anterior? | Campo `revoga_ids` com encadeamento completo de revogações |
| Todas as nomeações da Secretaria X no período | Ontologia de atos de pessoal unificando ~15 sinônimos em `tipo_ato_canonico` |
| Este cargo comissionado está disponível? | Índice de cargos com estado atual + histórico de ocupações |
| Quem foi nomeado/exonerado pela governadora vs secretário? | Campo `autoridade_assinante` classificado por categoria |

---

## 2. Arquitetura geral

```
Diário Oficial publicado (PDF/texto)
        │
        ▼
┌───────────────────────────────────┐
│     Pipeline de ingestão          │
│  Parser → Classificador → NER →   │
│  Detector de relações → Indexador │
└───────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────┐
│           ÍNDICE BASE: diario_materias           │
│  Documento original imutável por matéria         │
│  Nunca tem sua estrutura alterada                │
└─────────┬──────────┬──────────┬─────────────────┘
          │          │          │
          ▼          ▼          ▼
   idx_decretos  idx_pessoal  idx_cargos  [idx_novos...]
   (vigência)    (pessoal)    (estado)    (sem reindexar)
          │          │          │
          └────┬─────┘          │
               ▼                │
        Query Engine            │
     BM25 + kNN + filtros ◄─────┘
               │
               ▼
         Reranker (top-k)
               │
               ▼
     Claude — síntese contextual
               │
               ▼
    Resposta com fonte, data, decreto
```

---

## 3. Índice base — `diario_materias`

Este é o coração da arquitetura. Guarda o documento original de cada matéria exatamente como foi publicado. Sua estrutura **nunca muda** — é o ponto de partida de todos os índices derivados.

### Mapping

```json
{
  "mappings": {
    "properties": {
      "id":             { "type": "keyword" },
      "edicao":         { "type": "keyword" },
      "data_publicacao":{ "type": "date", "format": "yyyy-MM-dd" },
      "caderno":        { "type": "keyword" },
      "pagina":         { "type": "integer" },
      "tipo_materia":   { "type": "keyword" },
      "texto_completo": {
        "type": "text",
        "analyzer": "portuguese"
      },
      "texto_embedding": {
        "type": "knn_vector",
        "dimension": 1536,
        "method": {
          "name": "hnsw",
          "space_type": "cosinesimil",
          "engine": "faiss"
        }
      }
    }
  }
}
```

### Regra importante

Qualquer campo novo (ex: `tipo_ato_canonico`, `vigente`, `entidades`) **não vai para o índice base** — vai para o índice derivado correspondente. Isso garante que os derivados possam ser reconstruídos a qualquer momento sem perda.

---

## 4. Índices derivados

### 4.1 `idx_decretos` — Grafo de vigência

Resolve: *"Qual decreto está em vigor hoje sobre X?"*, *"Qual decreto revogou o Decreto Nº Y?"*

```json
{
  "mappings": {
    "properties": {
      "base_id":          { "type": "keyword" },
      "numero_decreto":   { "type": "keyword" },
      "data_assinatura":  { "type": "date" },
      "data_publicacao":  { "type": "date" },
      "ementa":           { "type": "text", "analyzer": "portuguese" },
      "texto_completo":   { "type": "text", "analyzer": "portuguese" },
      "texto_embedding":  { "type": "knn_vector", "dimension": 1536 },
      "vigente":          { "type": "boolean" },
      "revoga_ids":       { "type": "keyword" },
      "alterado_por_ids": { "type": "keyword" },
      "revogado_por_id":  { "type": "keyword" },
      "tipo_decreto":     { "type": "keyword" },
      "secretaria_ref":   { "type": "keyword" },
      "assinante":        { "type": "keyword" }
    }
  }
}
```

#### Lógica de vigência no pipeline

```python
# Padrões detectados no texto para atualização do grafo
PADROES_REVOGACAO = [
    r"fica(?:m)? revogad[oa]s?\s+o\s+Decreto\s+[Nn][ºo°]\s*([\d\.]+)",
    r"revogam-se\s+as\s+disposi[çc][õo]es\s+em\s+contr[áa]rio",
    r"ficam\s+revogados\s+os\s+Decretos\s+[Nn][ºo°]\s*([\d\.,\s]+)",
]

PADROES_ALTERACAO = [
    r"altera\s+o\s+art\.\s*\d+\s+do\s+Decreto\s+[Nn][ºo°]\s*([\d\.]+)",
    r"d[áa]\s+nova\s+reda[çc][ãa]o\s+ao\s+Decreto\s+[Nn][ºo°]\s*([\d\.]+)",
]

def processar_decreto(texto: str, numero: str) -> dict:
    revoga = extrair_numeros(texto, PADROES_REVOGACAO)
    altera = extrair_numeros(texto, PADROES_ALTERACAO)
    
    # Atualiza decretos anteriores como não vigentes
    for num in revoga:
        opensearch.update(
            index="idx_decretos",
            id=f"decreto_{num}",
            body={"doc": {"vigente": False, "revogado_por_id": numero}}
        )
    
    return {
        "vigente": True,
        "revoga_ids": revoga,
        "alterado_por_ids": altera,
    }
```

#### Consulta por vigência

```json
{
  "query": {
    "bool": {
      "must": [
        { "match": { "ementa": "licitação pregão eletrônico" } },
        { "term": { "vigente": true } }
      ],
      "filter": [
        { "range": { "data_publicacao": { "gte": "2019-01-01" } } }
      ]
    }
  }
}
```

---

### 4.2 `idx_pessoal` — Atos de pessoal

Resolve: *"Todas as nomeações da Secretaria de Saúde em 2023"*, *"Quem foi transferido pelo secretário da Educação?"*

```json
{
  "mappings": {
    "properties": {
      "base_id":              { "type": "keyword" },
      "nome_servidor":        { "type": "text", "analyzer": "portuguese",
                                "fields": { "keyword": { "type": "keyword" } } },
      "cpf_hash":             { "type": "keyword" },
      "matricula":            { "type": "keyword" },
      "cargo_origem":         { "type": "text", "analyzer": "portuguese" },
      "cargo_destino":        { "type": "text", "analyzer": "portuguese" },
      "secretaria_origem":    { "type": "keyword" },
      "secretaria_destino":   { "type": "keyword" },
      "tipo_ato_texto":       { "type": "keyword" },
      "tipo_ato_canonico":    { "type": "keyword" },
      "autoridade_assinante": { "type": "keyword" },
      "categoria_assinante":  { "type": "keyword" },
      "data_ato":             { "type": "date" },
      "data_publicacao":      { "type": "date" },
      "numero_decreto_ref":   { "type": "keyword" },
      "texto_embedding":      { "type": "knn_vector", "dimension": 1536 }
    }
  }
}
```

#### Ontologia de atos de pessoal

Esta é a peça central para resolver a fragmentação de vocabulário:

```python
ONTOLOGIA_ATOS = {
    # Nomeações
    "NOMEACAO": [
        "nomear", "nomeação", "nomeia", "nomeado",
        "designar para cargo", "designação para cargo em comissão",
        "investir", "provimento", "tomar posse"
    ],
    # Exonerações
    "EXONERACAO": [
        "exonerar", "exoneração", "exonerado", "dispensar",
        "dispensa", "dispensado", "desvincular", "desligamento",
        "a pedido", "ex officio", "vacância"
    ],
    # Transferências internas (secretários fazem isso)
    "TRANSFERENCIA": [
        "transferir", "transferência", "transferido", "remover",
        "remoção", "redistribuir", "redistribuição", "lotar",
        "lotação", "relotação", "deslocar", "movimentar"
    ],
    # Designações para função (sem mudar o cargo efetivo)
    "DESIGNACAO_FUNCAO": [
        "designar para função", "designar para responder",
        "designar para substituir", "designar interinamente",
        "responder pelo expediente", "substituição eventual"
    ],
    # Colocação à disposição
    "CESSAO": [
        "colocar à disposição", "ceder", "cessão",
        "requisitar", "requisição", "à disposição de"
    ],
    # Aposentadorias
    "APOSENTADORIA": [
        "aposentar", "aposentadoria", "aposentado",
        "inatividade", "invalidez", "compulsória"
    ]
}

def canonicalizar_ato(texto_ato: str) -> str:
    texto_lower = texto_ato.lower()
    for canonico, sinonimos in ONTOLOGIA_ATOS.items():
        if any(s in texto_lower for s in sinonimos):
            return canonico
    return "OUTROS"
```

#### Classificação da autoridade assinante

```python
HIERARQUIA_ASSINANTES = {
    "GOVERNADORA": [
        "governadora do estado", "governador do estado",
        "governo do estado de pernambuco"
    ],
    "SECRETARIO": [
        r"secretari[oa]\s+de\s+estado",
        r"secretari[oa]\s+executiv[oa]",
        r"secret[áa]ri[oa]\s+de\s+"
    ],
    "ORGAO_AUTONOMO": [
        "procurador", "defensor", "controlador"
    ]
}
```

#### Consulta exemplo — todas as movimentações de uma secretaria

```json
{
  "query": {
    "bool": {
      "filter": [
        {
          "bool": {
            "should": [
              { "term": { "secretaria_origem": "SECRETARIA DE SAUDE" } },
              { "term": { "secretaria_destino": "SECRETARIA DE SAUDE" } }
            ]
          }
        },
        {
          "terms": {
            "tipo_ato_canonico": ["NOMEACAO", "EXONERACAO", "TRANSFERENCIA"]
          }
        },
        {
          "range": {
            "data_publicacao": { "gte": "2019-01-01", "lte": "2026-12-31" }
          }
        }
      ]
    }
  },
  "size": 500,
  "sort": [{ "data_publicacao": "asc" }]
}
```

---

### 4.3 `idx_cargos` — Estado de cargos comissionados

Resolve: *"O cargo de Assessor Especial da SEFAZ está disponível?"*, *"Quem ocupa o cargo X atualmente?"*

```json
{
  "mappings": {
    "properties": {
      "id_cargo":           { "type": "keyword" },
      "nome_cargo":         { "type": "text", "analyzer": "portuguese",
                              "fields": { "keyword": { "type": "keyword" } } },
      "secretaria":         { "type": "keyword" },
      "nivel":              { "type": "keyword" },
      "simbolo":            { "type": "keyword" },
      "status_atual":       { "type": "keyword" },
      "ocupante_atual":     { "type": "keyword" },
      "decreto_criacao":    { "type": "keyword" },
      "data_ultima_atualizacao": { "type": "date" },
      "historico": {
        "type": "nested",
        "properties": {
          "servidor":       { "type": "keyword" },
          "tipo_ato":       { "type": "keyword" },
          "data_inicio":    { "type": "date" },
          "data_fim":       { "type": "date" },
          "decreto_ref":    { "type": "keyword" },
          "base_id":        { "type": "keyword" }
        }
      }
    }
  }
}
```

#### Lógica de atualização de estado

```python
def atualizar_cargo(cargo_id: str, ato: dict):
    cargo = opensearch.get(index="idx_cargos", id=cargo_id)
    historico = cargo["_source"].get("historico", [])
    
    if ato["tipo_ato_canonico"] == "NOMEACAO":
        # Fecha a ocupação anterior se existir
        if cargo["_source"]["status_atual"] == "OCUPADO":
            historico[-1]["data_fim"] = ato["data_ato"]
        
        # Abre nova ocupação
        historico.append({
            "servidor": ato["nome_servidor"],
            "tipo_ato": "NOMEACAO",
            "data_inicio": ato["data_ato"],
            "data_fim": None,
            "decreto_ref": ato["numero_decreto_ref"],
            "base_id": ato["base_id"]
        })
        novo_status = {"status_atual": "OCUPADO", "ocupante_atual": ato["nome_servidor"]}
    
    elif ato["tipo_ato_canonico"] == "EXONERACAO":
        if historico:
            historico[-1]["data_fim"] = ato["data_ato"]
        novo_status = {"status_atual": "DISPONIVEL", "ocupante_atual": None}
    
    opensearch.update(
        index="idx_cargos",
        id=cargo_id,
        body={"doc": {**novo_status, "historico": historico,
                      "data_ultima_atualizacao": ato["data_publicacao"]}}
    )
```

---

## 5. Pipeline de ingestão

### 5.1 Fluxo completo

```
PDF/texto do diário
      │
      ▼
1. Parser de matérias
   └── Separa por marcadores de seção
   └── Extrai: edição, data, caderno, página
      │
      ▼
2. Classificador de tipo
   └── Modelo: regex + LLM para casos ambíguos
   └── Saída: DECRETO | NOMEACAO_EXONERACAO | CONTRATO | EDITAL | OUTROS
      │
      ▼
3. NER — Extrator de entidades
   └── nome_servidor, cargo, secretaria, número_decreto
   └── Modelo: spaCy pt_core_news_lg + regras customizadas
      │
      ▼
4. Geração de embedding
   └── Modelo: text-embedding-3-small (OpenAI) ou multilingual-e5-large
   └── Chunking: matéria completa se < 512 tokens, sliding window se maior
      │
      ▼
5. Indexação no base
   └── POST diario_materias/_doc
      │
      ▼
6. Roteamento para derivados
   └── Se DECRETO → processar_decreto() → idx_decretos
   └── Se pessoal → canonicalizar_ato() → idx_pessoal + atualizar_cargo() → idx_cargos
```

### 5.2 Ingestão retroativa do acervo 2019–hoje

Para popular os índices derivados a partir do acervo já existente no índice base:

```python
from opensearchpy.helpers import scan

def reindexar_derivado(index_destino: str, processor_fn):
    """Lê o índice base e popula um derivado sem downtime."""
    query = {"query": {"match_all": {}}}
    
    for doc in scan(opensearch, index="diario_materias", query=query):
        try:
            resultado = processor_fn(doc["_source"])
            if resultado:
                opensearch.index(
                    index=index_destino,
                    id=resultado["id"],
                    body=resultado
                )
        except Exception as e:
            logger.error(f"Erro processando {doc['_id']}: {e}")
    
    logger.info(f"Reindexação de {index_destino} concluída.")

# Execução
reindexar_derivado("idx_decretos", processar_decreto)
reindexar_derivado("idx_pessoal",  processar_pessoal)
reindexar_derivado("idx_cargos",   processar_cargos)
```

### 5.3 Criação de novo índice no futuro (sem reindexar os outros)

```python
# Exemplo: novo índice de licitações criado 2 anos depois
def criar_indice_licitacoes():
    # 1. Cria o índice com o novo mapping
    opensearch.indices.create(index="idx_licitacoes", body=MAPPING_LICITACOES)
    
    # 2. Pipeline lê apenas matérias de tipo EDITAL/LICITACAO do índice base
    query = {"query": {"term": {"tipo_materia": "LICITACAO"}}}
    reindexar_derivado("idx_licitacoes", processar_licitacao)
    
    # Índices idx_decretos, idx_pessoal, idx_cargos não são tocados
```

---

## 6. Estratégia de busca híbrida

### 6.1 Query combinada BM25 + kNN

```python
def buscar_hibrida(
    query_text: str,
    query_embedding: list,
    filtros: dict,
    index: str,
    size: int = 50
) -> list:
    
    body = {
        "size": size,
        "query": {
            "bool": {
                "should": [
                    {
                        "match": {
                            "texto_completo": {
                                "query": query_text,
                                "boost": 1.0
                            }
                        }
                    },
                    {
                        "knn": {
                            "texto_embedding": {
                                "vector": query_embedding,
                                "k": size,
                                "boost": 1.5
                            }
                        }
                    }
                ],
                "filter": construir_filtros(filtros)
            }
        }
    }
    
    return opensearch.search(index=index, body=body)["hits"]["hits"]


def construir_filtros(filtros: dict) -> list:
    f = []
    if filtros.get("secretaria"):
        f.append({"term": {"secretaria": filtros["secretaria"]}})
    if filtros.get("tipo_ato_canonico"):
        f.append({"terms": {"tipo_ato_canonico": filtros["tipo_ato_canonico"]}})
    if filtros.get("vigente") is not None:
        f.append({"term": {"vigente": filtros["vigente"]}})
    if filtros.get("data_inicio"):
        f.append({"range": {"data_publicacao": {
            "gte": filtros["data_inicio"],
            "lte": filtros.get("data_fim", "now")
        }}})
    return f
```

### 6.2 Reranker com Cross-Encoder (opcional, alta precisão)

```python
from sentence_transformers import CrossEncoder

reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")

def rerankar(query: str, docs: list, top_k: int = 20) -> list:
    pares = [(query, doc["_source"]["texto_completo"][:512]) for doc in docs]
    scores = reranker.predict(pares)
    docs_scored = sorted(zip(docs, scores), key=lambda x: x[1], reverse=True)
    return [doc for doc, _ in docs_scored[:top_k]]
```

---

## 7. Integração com Claude (Anthropic)

### 7.1 Prompt de síntese

```python
import anthropic

client = anthropic.Anthropic()

SYSTEM_PROMPT = """Você é um especialista em legislação estadual de Pernambuco.
Responda perguntas sobre os Diários Oficiais do Estado com base nos documentos fornecidos.

Regras:
- Cite sempre a edição do diário, data e número do decreto/ato
- Se um decreto foi revogado, informe claramente e indique qual o substituiu
- Para nomeações/exonerações, indique quem assinou o ato (governadora ou secretário)
- Se não encontrar informação nos documentos, diga explicitamente
- Seja preciso com datas e números"""

def sintetizar_com_claude(pergunta: str, documentos: list) -> str:
    contexto = formatar_documentos(documentos)
    
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=2000,
        system=SYSTEM_PROMPT,
        messages=[{
            "role": "user",
            "content": f"""Documentos encontrados:\n\n{contexto}\n\n
            Pergunta: {pergunta}"""
        }]
    )
    return response.content[0].text


def formatar_documentos(docs: list) -> str:
    partes = []
    for i, doc in enumerate(docs, 1):
        s = doc["_source"]
        partes.append(
            f"[{i}] Edição {s.get('edicao')} — {s.get('data_publicacao')}\n"
            f"Tipo: {s.get('tipo_materia')} | "
            f"Decreto: {s.get('numero_decreto', 'N/A')} | "
            f"Vigente: {s.get('vigente', 'N/A')}\n"
            f"{s.get('texto_completo', '')[:800]}\n"
        )
    return "\n---\n".join(partes)
```

### 7.2 Fluxo completo de uma pergunta

```python
def responder_pergunta(pergunta: str, filtros: dict = {}) -> dict:
    # 1. Gera embedding da pergunta
    embedding = gerar_embedding(pergunta)
    
    # 2. Determina qual(is) índice(s) consultar
    indices = rotear_indice(pergunta)  # heurística ou LLM classifier
    
    # 3. Busca híbrida em cada índice relevante
    candidatos = []
    for idx in indices:
        resultados = buscar_hibrida(
            query_text=pergunta,
            query_embedding=embedding,
            filtros=filtros,
            index=idx,
            size=50
        )
        candidatos.extend(resultados)
    
    # 4. Reranking
    top_docs = rerankar(pergunta, candidatos, top_k=15)
    
    # 5. Síntese com Claude
    resposta = sintetizar_com_claude(pergunta, top_docs)
    
    return {
        "resposta": resposta,
        "fontes": [
            {
                "edicao": d["_source"].get("edicao"),
                "data": d["_source"].get("data_publicacao"),
                "decreto": d["_source"].get("numero_decreto"),
                "score": d["_score"]
            }
            for d in top_docs
        ]
    }
```

### 7.3 Casos especiais: consultas agregadas

Para perguntas que precisam de todos os registros (não apenas os mais relevantes):

```python
def listar_decretos_vigentes(tema: str, secretaria: str = None) -> dict:
    """Retorna TODOS os decretos vigentes sobre um tema — não usa top-k."""
    
    filtros = {"vigente": True}
    if secretaria:
        filtros["secretaria_ref"] = secretaria
    
    # Busca completa sem limite de k
    body = {
        "query": {
            "bool": {
                "must": [{"match": {"ementa": tema}}],
                "filter": construir_filtros(filtros)
            }
        },
        "sort": [{"data_publicacao": "desc"}],
        "size": 1000  # ou usar scroll para volumes maiores
    }
    
    resultados = opensearch.search(index="idx_decretos", body=body)
    total = resultados["hits"]["total"]["value"]
    docs = resultados["hits"]["hits"]
    
    # Claude analisa a lista completa
    resumo = sintetizar_com_claude(
        f"Liste e explique os {total} decretos vigentes sobre '{tema}'",
        docs
    )
    
    return {"total": total, "resumo": resumo, "decretos": docs}
```

---

## 8. Criação de novos índices ao longo do tempo

### Quando NÃO precisa reindexar

- Novo índice derivado (licitações, contratos, convênios, portarias)
- Novo campo adicionado via `dynamic mapping` em documentos futuros
- Novo alias ou view sobre índices existentes

### Quando PRECISA reindexar (apenas o derivado afetado)

- Mudança de tipo de campo existente (ex: `text` → `keyword`)
- Adição de campo vetorial em documentos já indexados
- Mudança do modelo de embedding (dimensão diferente)

### Procedimento de reindexação sem downtime

```python
def reindexar_sem_downtime(index_atual: str, novo_mapping: dict, processor_fn):
    novo_index = f"{index_atual}_v2"
    
    # 1. Cria novo índice com mapping atualizado
    opensearch.indices.create(index=novo_index, body=novo_mapping)
    
    # 2. Popula a partir do índice base (o atual continua respondendo)
    reindexar_derivado(novo_index, processor_fn)
    
    # 3. Quando pronto, troca o alias
    opensearch.indices.update_aliases(body={
        "actions": [
            {"remove": {"index": index_atual, "alias": index_atual.split("_v")[0]}},
            {"add":    {"index": novo_index,   "alias": index_atual.split("_v")[0]}}
        ]
    })
    
    # 4. Remove o índice antigo após validação
    # opensearch.indices.delete(index=index_atual)
    print(f"Switch completo: {index_atual} → {novo_index}")
```

---

## 9. Roadmap de implementação

### Fase 1 — Fundação (semanas 1–3)

- [ ] Revisar e validar o índice base existente (`diario_materias`)
- [ ] Implementar o classificador de tipo de matéria
- [ ] Criar `idx_decretos` e popular com o acervo 2019–hoje
- [ ] Validar grafo de vigência em amostra de 100 decretos

### Fase 2 — Pessoal (semanas 4–6)

- [ ] Implementar ontologia de atos de pessoal
- [ ] Criar `idx_pessoal` e popular com o acervo
- [ ] Validar classificação de autoridade assinante
- [ ] Criar `idx_cargos` com estado atual de cargos comissionados

### Fase 3 — Busca e síntese (semanas 7–9)

- [ ] Implementar query engine híbrida BM25 + kNN
- [ ] Integrar reranker (Cross-Encoder)
- [ ] Implementar integração com Claude API
- [ ] Testes de qualidade: recall e precisão por tipo de pergunta

### Fase 4 — Interface e produção (semanas 10–12)

- [ ] API REST (FastAPI) para receber perguntas e retornar respostas
- [ ] Interface de chat web (React ou Streamlit)
- [ ] Pipeline de atualização diária automática
- [ ] Monitoramento e alertas

---

## 10. Estimativas de volume

| Índice | Registros estimados | Tamanho estimado |
|---|---|---|
| `diario_materias` (base) | ~500 mil matérias | ~15–25 GB |
| `idx_decretos` | ~30–50 mil decretos | ~3–5 GB |
| `idx_pessoal` | ~200–400 mil atos | ~8–12 GB |
| `idx_cargos` | ~5–10 mil cargos | ~500 MB |

---

## 11. Dependências Python

```txt
opensearch-py>=2.4.0
anthropic>=0.25.0
sentence-transformers>=2.6.0
spacy>=3.7.0
pt-core-news-lg>=3.7.0      # python -m spacy download pt_core_news_lg
pdfplumber>=0.10.0
fastapi>=0.110.0
uvicorn>=0.29.0
pydantic>=2.0.0
tenacity>=8.2.0
loguru>=0.7.0
```

---

## 12. Variáveis de ambiente

```bash
OPENSEARCH_HOST=https://seu-opensearch:9200
OPENSEARCH_USER=admin
OPENSEARCH_PASS=sua_senha
ANTHROPIC_API_KEY=sk-ant-...
EMBEDDING_MODEL=text-embedding-3-small
OPENAI_API_KEY=sk-...          # apenas se usar embeddings da OpenAI
RERANKER_MODEL=cross-encoder/ms-marco-MiniLM-L-6-v2
LOG_LEVEL=INFO
```

---

*Documento gerado para uso com Claude Code na implementação.*  
*Próximo passo: abrir o Claude Code com este arquivo como referência e começar pela Fase 1.*

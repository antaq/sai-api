# API Pública de Dados do SAI

Dados abertos dos investimentos em instalações portuárias acompanhados pela
**ANTAQ** — Agência Nacional de Transportes Aquaviários.

O SAI (Sistema de Acompanhamento de Investimentos) registra as obrigações de
investimento previstas nos contratos de arrendamento e nas adesões de
terminais autorizados, e acompanha sua execução. Esta API publica esses dados.

📖 **Documentação interativa:** <https://antaq.github.io/sai-api/>

| | |
|---|---|
| **Produção** | `https://prd-apex.antaq.gov.br/ords/sfc_sai/dados` |
| **Autenticação** | nenhuma |
| **Formatos** | JSON, CSV, XLSX |
| **Especificação** | [`openapi.yaml`](openapi.yaml) (OpenAPI 3.0.3) |
| **Licença dos dados** | [Dados Abertos gov.br](https://www.gov.br/governodigital/pt-br/dados-abertos) |

## Comece por aqui

```bash
# A API se descreve: recursos, parâmetros e valores válidos
curl https://prd-apex.antaq.gov.br/ords/sfc_sai/dados/

# Resumo consolidado da carteira
curl https://prd-apex.antaq.gov.br/ords/sfc_sai/dados/consolidado/resumo

# Contratos de São Paulo em Excel
curl -O -J "https://prd-apex.antaq.gov.br/ords/sfc_sai/dados/contratos?uf=SP&formato=xlsx"

# Tudo, em um ZIP de CSVs, com LEIAME e dicionário de dados
curl -O -J https://prd-apex.antaq.gov.br/ords/sfc_sai/dados/download/completo
```

## O que é publicado

Os **contratos** aparecem integralmente: dados cadastrais, localização e o
investimento global previsto. Essa informação é produzida pela ANTAQ e é
pública desde a celebração do contrato.

Os dados de **execução** — quanto foi investido, qual o percentual físico
realizado — só são publicados depois que o acompanhamento foi **concluído pela
Fiscalização**. Números ainda em análise não saem, porque ainda não foram
verificados.

> [!IMPORTANT]
> A carteira aparece inteira, mas parte dos contratos vem com
> `possui_execucao_validada = 0` e campos de execução nulos. Use
> `cobertura_execucao_pct`, em `/consolidado/resumo`, para saber que fração da
> carteira já passou pela Fiscalização. Dividir execução por investimento
> global sem esse cuidado subestima o percentual executado.

Esta API **não expõe dado pessoal**: nomes, CPF, e-mail e telefone de
representantes legais não estão em nenhum endpoint, assim como observações e
pareceres internos de análise. Empresas aparecem por razão social e CNPJ.

## Endpoints

| Recurso | Descrição |
|---|---|
| `GET /` | documento de descoberta |
| `GET /contratos` | lista de contratos |
| `GET /contratos/{id}` | ficha completa |
| `GET /contratos/{id}/acompanhamentos` | execução validada do contrato |
| `GET /contratos/{id}/indices` | índices e vigências do contrato |
| `GET /acompanhamentos` | todos os acompanhamentos validados |
| `GET /consolidado/resumo` | totais da carteira |
| `GET /consolidado/por/{dimensao}` | agregação por região, UF, empresa, porto, situação, tipo de instalação, perfil de carga, índice ou ano |
| `GET /consolidado/evolucao-mensal` | série mensal de execução |
| `GET /consolidado/evolucao-carteira` | entrada de contratos ao longo do tempo |
| `GET /consolidado/faixas-execucao` | distribuição por faixa de % executado |
| `GET /consolidado/reidi` | contratos com REIDI |
| `GET /mapa` | pontos georreferenciados |
| `GET /indices` | índices econômicos disponíveis |
| `GET /indices/{sigla}` | série histórica de um índice |
| `GET /dominios` | catálogos de códigos |
| `GET /metadados` | dicionário de dados |
| `GET /download/completo` | ZIP com todos os conjuntos |

Parâmetros, campos e exemplos de resposta estão na
[documentação interativa](https://antaq.github.io/sai-api/).

## Valores corrigidos

Cada contrato tem cláusula própria de reajuste, com um ou mais índices
(IPCA, IGP-M, INPC, INCC) e períodos de vigência. O parâmetro
`data_reajuste` define a data-alvo da correção; omitido, vale o mês corrente.

Todo valor monetário vem decomposto, para que a conta possa ser refeita:

```
investimento_global_nominal     valor original, na data-base contratual
investimento_global_data_base   data-base do valor nominal
investimento_global_indice      sigla do índice aplicado
investimento_global_fator       multiplicador acumulado (6 casas)
investimento_global_corrigido   nominal × fator (2 casas)
investimento_global_correcao    diagnóstico do cálculo
```

O diagnóstico existe porque fator igual a 1 é ambíguo — pode ser "contrato sem
reajuste" ou "não foi possível corrigir". O valor
`APLICADA_ATE_ULTIMA_COMPETENCIA` é o **caso normal** quando `data_reajuste`
não é informada, já que as séries de índices são publicadas com defasagem; a
competência efetivamente usada consta em `metadados.ultima_competencia_indices`,
presente em toda resposta. A lista completa está em `/dominios`.

## Paginação e exportações

Respostas JSON trazem no máximo **500** registros por página (padrão 25).
Percorra somando `returned_count` ao `offset` enquanto `has_more` for `true`.

```python
import requests
BASE = "https://prd-apex.antaq.gov.br/ords/sfc_sai/dados"
linhas, offset = [], 0
while True:
    p = requests.get(f"{BASE}/contratos", params={"limit": 500, "offset": offset}).json()
    linhas += p["items"]
    if not p["has_more"]:
        break
    offset += p["returned_count"]
```

**As exportações não paginam.** `formato=csv` e `formato=xlsx` devolvem o
recorte inteiro, independentemente de `limit` — normalmente é o caminho mais
simples:

```python
import pandas as pd
df = pd.read_csv(f"{BASE}/contratos?formato=csv", sep=";")
```

O CSV sai em UTF-8 com BOM, separador `;` e textos entre aspas: abre no Excel
em português sem etapa de importação. JSON, CSV e XLSX saem da **mesma
consulta**, então o arquivo baixado é exatamente o recorte visto no JSON.

## Conteúdo deste repositório

```
index.html              documentação interativa (GitHub Pages)
openapi.yaml            especificação OpenAPI 3.0.3
vendor/swagger-ui/      Swagger UI hospedado localmente, com SRI
```

O Swagger UI é servido deste repositório, e não de CDN, com verificação de
integridade (`integrity="sha384-..."`): a página não depende de terceiros para
carregar nem para funcionar, e nenhum dado de navegação sai para fora.

A implementação (pacotes PL/SQL, visões e o módulo ORDS) é mantida no
repositório interno do SAI.

## Contato

Gerência de Planejamento e Inteligência da Fiscalização — ANTAQ/SFC
📧 [gpf@antaq.gov.br](mailto:gpf@antaq.gov.br)

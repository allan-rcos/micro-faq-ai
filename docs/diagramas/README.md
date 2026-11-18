# Diagramas

Índice: [← Índice da documentação](../README.md)

Fontes PlantUML dos diagramas da documentação. Cada `.puml` gera o `.svg` de mesmo nome, que é o arquivo referenciado pelos documentos; o estilo comum fica em `_tema.iuml`.

| Diagrama | Usado em |
|----------|----------|
| [`01_ciclo_de_aprendizado`](01_ciclo_de_aprendizado.puml) | [Visão geral §3](../01_visao_geral.md#3-ciclo-de-aprendizado) |
| [`02_recursos_oci`](02_recursos_oci.puml) | [Infraestrutura §1](../02_infraestrutura.md#1-recursos) |
| [`04_pipeline_normalizacao`](04_pipeline_normalizacao.puml) | [Normalização §1](../04_normalizacao.md#1-etapas) |
| [`06_arquitetura_modelo`](06_arquitetura_modelo.puml) | [Treinamento §5.1](../06_treinamento.md#51-arquitetura) |
| [`10_estados_pergunta`](10_estados_pergunta.puml) | [Fluxo §5.1](../10_fluxo_de_resposta.md#51-estados-de-uma-pergunta) |

## Gerar

Depois de editar um `.puml`, gere de novo os SVGs e versione os dois juntos:

```bash
plantuml -tsvg -failfast2 docs/diagramas/*.puml
```

Requer PlantUML com Graphviz (`dot`), usado no diagrama de estados e no de recursos.

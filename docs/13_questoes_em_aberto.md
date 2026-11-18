# 13 — Questões em aberto

Índice: [← Índice](README.md) | [← Ambiente local](12_ambiente_local.md) | [Próximo: Changelog →](changelog/README.md)

Riscos que só a implementação e o uso vão confirmar. Cada item aponta a seção que muda se ele se concretizar; quando isso acontecer, a decisão vai para o documento dono, o item sai daqui e, se substituir algo já descrito, ganha uma linha no [changelog](changelog/README.md).

## 1. Riscos a acompanhar

| # | Risco | Como acompanhar | Se acontecer |
|---|-------|-----------------|--------------|
| 1 | Perguntas fora de todos os tópicos, mas com palavras em comum com algum, ainda recebendo probabilidade alta na sigmoide | Separar um conjunto de perguntas reais fora de escopo e medir a confiança delas a cada treino | Mais exemplos de `_conversa` e de perguntas fora de escopo como negativos; subir `faixas.alta` |
| 2 | O contrato do grafo, preso às limitações do gonnx (opset 13, operadores parciais, [Carga do modelo §5.1](08_carga_do_modelo.md#51-runtime-onnx)), travar uma mudança futura de arquitetura | O teste de conformidade roda as duas implementações a cada mudança do modelo | Aceitar que o gonnx deixa de ser alternativa e liberar o grafo só para o ONNX Runtime, que executa opset 17 e formas dinâmicas |
| 3 | Presença *digitando* não chegar de forma confiável em grupos grandes | Medir no grupo autônomo real | A rajada passa a fechar só por `espera_digitando` sem mensagens e `espera_maxima` |
| 4 | Diferenças de minúsculas entre Go e Python em caracteres raros | Casos em `testdata/normalizacao.jsonl` ([Normalização §6](04_normalizacao.md#6-paridade-entre-go-e-python)) | Tabela própria de minúsculas compartilhada pelos dois lados |
| 5 | Conversão do `cc.pt.300.bin` exigir mais de 16 GB de RAM na máquina de build | Rodar a conversão uma vez | Converter a partir do `cc.pt.300.vec` (só vetores de palavra) e dos *buckets* lidos em blocos |
| 6 | Uma atualização do Alpine trazer um `onnxruntime` cuja API não combina com o binding fixado | Teste de conformidade numa imagem da nova versão antes de atualizar a VM | Ajustar a versão do binding ([Carga do modelo §5.2](08_carga_do_modelo.md#52-onnx-runtime-oficial-em-uso)) ou subir o build com `-tags gonnx` enquanto isso |
| 7 | A sessão do WhatsApp no NoSQL ([Bot §1.1](09_bot_grupos.md#11-sessão-na-tabela-única)): latência por mensagem em grupos movimentados, ou uma mudança nas interfaces de `store` do whatsmeow | Medir o tempo de decifração por mensagem; rodar o teste de `sessaowa` a cada atualização da biblioteca | Manter a versão do whatsmeow até adaptar `sessaowa`; se a latência pesar, agrupar as gravações de uma mesma mensagem numa única operação em lote do NoSQL |
| 8 | O treino não caber em 1 GB de RAM mesmo com swap, ou levar mais que um dia | Acompanhar pico de memória e duração no log de cada treino | Reduzir `modelo.tamanho_lote`; em último caso, trocar a VM `treino` por uma A1 com mais memória |

Próximo: [Changelog →](changelog/README.md)

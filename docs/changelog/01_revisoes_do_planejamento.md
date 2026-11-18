# 01 — Revisões do planejamento

Índice: [← Changelog](README.md)

Decisões trocadas durante o planejamento, em 17/11/2026, depois da primeira versão desta documentação.

## 1. Modelo e classificação

| Era | Ficou | Por quê | Descrito em |
|-----|-------|---------|-------------|
| Saída softmax, probabilidades somando 1 | Sigmoide por tópico, com entropia cruzada binária | Uma mensagem fora de todos os tópicos precisa poder ter confiança baixa em todos | [Treinamento §5.1](../06_treinamento.md#51-arquitetura) |
| Só tópicos de dúvida no modelo | Tópicos reservados `_cumprimento`, `_negacao` e `_conversa`, treinados como os demais | O bot lê todas as mensagens do grupo autônomo e precisa reconhecer o que não é dúvida | [Dados §1.2](../05_dados.md#12-tópicos-reservados) |
| Runtime ONNX gonnx, binário Go estático sem CGO | ONNX Runtime oficial via `onnxruntime_go` (CGO, lib do Alpine), com o gonnx mantido como alternativa por *build tag* atrás de uma interface | ≈ 11× mais rápido na média e sem as pausas de dezenas de ms do gonnx; implementação de referência | [Carga do modelo §5](../08_carga_do_modelo.md#51-runtime-onnx) |
| ONNX opset 17, número de frases dinâmico, atenção com `ReduceSum` | Opset 13, entrada fixa de 8 frases, atenção com `MatMul` + `Transpose` | Limites do gonnx encontrados no teste em contêiner; mantidos para que o mesmo pacote rode nos dois runtimes | [Carga do modelo §5.1](../08_carga_do_modelo.md#51-runtime-onnx) |

## 2. Comportamento do bot

| Era | Ficou | Por quê | Descrito em |
|-----|-------|---------|-------------|
| Normalização em etapas, cada uma relendo o texto, com Aho-Corasick por biblioteca | Uma passada por caractere, janela de gírias e implementação própria, sem expressões regulares | Menos trabalho por mensagem sob carga | [Normalização §1](../04_normalizacao.md#1-etapas) |
| Letras repetidas não tratadas | Três ou mais letras iguais viram uma | — | [Normalização §3](../04_normalizacao.md#3-letras-repetidas-e-espaços) |
| Palpites ao administrador só com identificadores | Lista com o exemplo canônico de cada palpite | O administrador não precisa saber o que cada identificador representa | [Fluxo §6.1](../10_fluxo_de_resposta.md#61-caso) |
| Filtro de "aparência de pergunta" (`?` ou palavra interrogativa) no modo autônomo | Toda mensagem é classificada; cumprimentos e conversa caem nos tópicos reservados | — | [Bot §2.2](../09_bot_grupos.md#22-autônomo) |
| Negativa detectada por lista de termos na configuração | Negativa é o tópico treinado `_negacao` | Cresce com exemplos, como as substituições | [Fluxo §5](../10_fluxo_de_resposta.md#5-resposta-negativa) |
| Alternativas que expiravam abriam caso | Expiram em silêncio | — | [Fluxo §4](../10_fluxo_de_resposta.md#4-confiança-média-alternativas) |
| Grupo fechado por inundação reabria após um tempo fixo (`reabrir_apos`) | Reabre quando todas as perguntas capturadas tiveram a primeira saída | — | [Bot §2.2](../09_bot_grupos.md#22-autônomo) |
| Sem modelo, toda pergunta de todo grupo ia aos administradores | Sem modelo, só os grupos de marcação são processados | Sem `_conversa`, o grupo autônomo viraria um caso por mensagem | [Carga do modelo §4](../08_carga_do_modelo.md#4-troca) |

## 3. Dados e infraestrutura

| Era | Ficou | Por quê | Descrito em |
|-----|-------|---------|-------------|
| Treino sempre que o JSON mudava (cron a cada 30 minutos) | Verificação diária; treino só depois de `agenda.intervalo_dias` (30) e com dados novos | Acumular exemplos e não sobrecarregar a VM de treino | [Treinamento §1](../06_treinamento.md#1-disparo) |
| Estado do último treino em arquivo local (`estado.json`) | O último pacote publicado é o registro | VM sem estado | [Treinamento §1](../06_treinamento.md#1-disparo) |
| Duas VMs A1 arm64 (bot 1 OCPU/2 GB, treino 2 OCPU/8 GB) | Bot em A1 1 OCPU/12 GB (arm64); treino em E2.1.Micro 1 GB (x86_64) | VMs disponíveis; o treino pode ser lento | [Infraestrutura §2](../02_infraestrutura.md#2-máquinas-virtuais) |
| Regra de ciclo de vida apagando pacotes com mais de 30 dias | Últimos `publicacao.modelos_no_bucket` no bucket; todo pacote e seu JSON em release do repositório, com Brotli | Com treino mensal, a regra por idade deixaria o bucket sem modelo anterior | [Pacote do modelo §6](../07_pacote_modelo.md#6-retenção-e-backup) |
| Só *instance principals* | Modo `instancia` ou `arquivo` (chave de API) | Rodar fora da OCI e consultar de outros lugares | [Infraestrutura §5](../02_infraestrutura.md#5-identidade-e-permissões) |
| Sessão do WhatsApp em SQLite local (`sessao.db`) | Itens na tabela única, por implementação própria das interfaces de `store` do whatsmeow | Sem SQLite; VM sem estado | [Bot §1.1](../09_bot_grupos.md#11-sessão-na-tabela-única) |
| Configuração em JSON | TOML | — | [Configuração](../11_configuracao.md) |
| Chave `id` única com prefixo de tipo | `pk` + `sk` em todos os itens | Padronização | [Dados §1](../05_dados.md#1-tabela-única-nosql) |
| Cache em memória dos tópicos | Leitura por chave a cada uso | Leitura barata; memória fica para os vetores | [Dados §1](../05_dados.md#1-tabela-única-nosql) |
| `topic_id` de 6 caracteres base32 | nanoID de 8 caracteres, sempre gerado pelo bot | — | [Dados §1.1](../05_dados.md#11-tópico) |
| Tabela `topicos` no NoSQL e casos em SQLite local (`casos.db`) | Tabela única `micro_faq` (*single table design*) com tópicos e casos; TTL encerra casos | Nenhuma análise é feita sobre a tabela; os casos sobrevivem à perda da VM | [Dados §1](../05_dados.md#1-tabela-única-nosql) |
| Risco: Alpine sem imagem oficial na OCI | Removido | As VMs Alpine já existem | [Infraestrutura §2](../02_infraestrutura.md#2-máquinas-virtuais) |
| Risco: runtime ONNX em Go puro não executar o grafo | Validado em experimento local | — | [Carga do modelo §5.1](../08_carga_do_modelo.md#51-runtime-onnx) |

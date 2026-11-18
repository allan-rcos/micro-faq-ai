# 08 — Carga do modelo no bot

Índice: [← Índice](README.md) | [← Pacote do modelo](07_pacote_modelo.md) | [Próximo: Bot e modos de grupo →](09_bot_grupos.md)

Este documento descreve como o bot descobre um pacote novo, verifica, testa e troca o modelo em execução sem reiniciar, e como faz a inferência de uma mensagem. O formato do pacote está em [Pacote do modelo](07_pacote_modelo.md).

## 1. Descoberta

Na inicialização e depois a cada `modelo.verificar_a_cada` ([Configuração §1](11_configuracao.md#1-bot)), o bot lista o prefixo `modelos/` do bucket e escolhe o último nome em ordem lexicográfica cujo `major` ele implementa ([Pacote §4](07_pacote_modelo.md#4-nome-no-bucket)). Se esse nome é o do pacote em uso, ou um já recusado nesta execução, não faz nada.

## 2. Verificação

O pacote baixado passa, em ordem, por:

1. SHA-256 do arquivo igual ao metadado `opc-meta-sha256`.
2. Desserialização do Protocol Buffers.
3. `versao.major` e `versao_normalizacao` suportados pelo binário.
4. Coerência interna: `len(dicionario.vetores) = len(palavras) × dimensao × 4`; o ONNX carrega, recebe `[1, 8, dimensao]` e devolve `[1, 1, len(topicos)]`.
5. Os três tópicos reservados estão em `topicos` ([Dados §1.2](05_dados.md#12-tópicos-reservados)).
6. Todo `topic_id` de `topicos` existe na tabela NoSQL ([Dados §1](05_dados.md#1-tabela-única-nosql)).

## 3. Teste

Com o pacote verificado, o bot monta um classificador candidato (normalizador com as substituições do pacote + mini-dicionário + ONNX) e classifica o **exemplo canônico** de cada tópico do mapa. O teste passa se:

- o tópico previsto é o próprio tópico do exemplo em pelo menos `modelo.teste_acerto_minimo` % dos tópicos; e
- nenhuma classificação falha com erro.

O exemplo canônico é escrito pelos administradores e não está necessariamente no JSON de treinamento, então o teste mede o modelo sobre frases que ele pode não ter visto.

## 4. Troca

- O classificador em uso fica num `atomic.Pointer`. Passando no teste, o candidato substitui o atual com uma única troca de ponteiro: classificações em andamento terminam com o modelo antigo, as seguintes usam o novo.
- O bot envia ao grupo de administração uma linha com a versão nova, o número de tópicos e as métricas do pacote.
- Falhando qualquer etapa de §2 ou §3, o modelo em uso continua, o nome do pacote entra na lista de recusados desta execução e o grupo de administração recebe o motivo.

**Sem nenhum pacote válido** (primeira subida, bucket vazio), o bot conecta ao WhatsApp normalmente, mas só processa os grupos do modo marcação: toda pergunta capturada ali é tratada como confiança baixa e vai ao grupo de administração. Os grupos autônomos ficam sem leitura até o primeiro pacote, porque sem o `_conversa` treinado toda mensagem viraria um caso. É assim que os primeiros tópicos nascem ([Fluxo §6](10_fluxo_de_resposta.md#6-grupo-de-administração)).

## 5. Inferência

Para uma pergunta (uma ou várias mensagens):

1. **Normalização** com o autômato do pacote ([Normalização](04_normalizacao.md)) → frases de tokens.
2. **Índices**: cada token vira seu índice no mini-dicionário, ou `<unk>`.
3. **Vetores de frase**: média dos vetores conhecidos e normalização L2, completando até 8 linhas, mais a máscara ([Treinamento §4](06_treinamento.md#4-representação-vetorial)). Feito em Go com laços sobre `[]float32`.
4. **ONNX**: `sentencas` e `mascara` → `probabilidades` `[1, 1, K]`.
5. **Resultado**: os pares `(topicos[i], p_i)` em ordem decrescente de probabilidade. O que se faz com eles é de [Fluxo §1](10_fluxo_de_resposta.md#1-decisão).

Se nenhuma frase tem token conhecido, a mensagem não é classificável e é tratada como `_conversa` ([Fluxo §2](10_fluxo_de_resposta.md#2-tópicos-reservados)).

### 5.1. Runtime ONNX

O runtime que executa o ONNX é uma dependência substituível. O bot usa o **ONNX Runtime oficial** (§5.2) e mantém o **gonnx** (§5.3) como alternativa pronta; a troca entre eles muda um *build tag*, não o código do bot nem o pacote do modelo.

**Desacoplamento.** Só o pacote `bot/internal/inferencia` conhece um runtime ONNX. O resto do bot — carga, teste, classificador — depende apenas desta interface:

```go
package inferencia

// Runtime abre sessões a partir dos bytes do campo onnx do pacote do modelo.
type Runtime interface {
	Abrir(onnx []byte) (Sessao, error)
}

// Sessao executa o grafo de Treinamento §6.
type Sessao interface {
	// sentencas: 8 × dimensao valores, linha a linha; mascara: 8 valores (1 = frase válida).
	// Devolve as K probabilidades, na ordem de Pacote.topicos.
	Executar(sentencas, mascara []float32) ([]float32, error)
	Fechar() error
}
```

| Regra | Por quê |
|-------|---------|
| Nenhum outro pacote importa `onnxruntime_go` ou `gonnx`, nem tipos deles atravessam a interface | A troca de runtime não toca o resto do bot |
| Cada implementação fica num subpacote com *build tag* próprio — `inferencia/ort` (`-tags ort`, padrão) e `inferencia/gonnx` (`-tags gonnx`) —, e só a escolhida entra no binário | O build com gonnx continua sem CGO e estático; o com ORT não carrega código do gonnx |
| As duas implementações passam no mesmo teste de conformidade, `inferencia/conformidade_test.go`, que roda um ONNX de referência e compara com as saídas do PyTorch (tolerância 1e-5) | Garante que a alternativa continua funcionando mesmo sem estar em uso |
| O grafo exportado fica no subconjunto que **os dois** runtimes executam (abaixo) | Um mesmo pacote do modelo serve a qualquer um dos dois, sem novo treino |

**Contrato do grafo.** O gonnx é o mais restrito dos dois, então as regras dele valem para toda exportação ([Treinamento §6](06_treinamento.md#6-exportação-onnx)):

| Regra | Motivo |
|-------|--------|
| Opset 13 | É o único opset que o gonnx implementa |
| Sem `ReduceSum`, `Squeeze` e `Unsqueeze` | `ReduceSum` não existe no gonnx; o `Squeeze` dele remove todas as dimensões de tamanho 1, não só a pedida. A soma ponderada da atenção é feita como `MatMul` de `[1, 1, 8]` por `[1, 8, 256]` |
| Entrada de tamanho fixo (8 frases) | Com uma dimensão de tamanho 1 no lugar das frases, o `MatMul` do gonnx falha |

O grafo resultante usa só `Add`, `Constant`, `Div`, `MatMul`, `Mul`, `Sigmoid`, `Softmax`, `Sub`, `Tanh` e `Transpose`. A verificação de §2.4 recusa um pacote cujo grafo o runtime em uso não execute.

Os experimentos citados nesta seção (exportação pelo PyTorch, execução nos dois runtimes e medição) ficam na pasta `spikes/`, local e fora do controle de versão; o teste de conformidade de §5.1 é a versão permanente deles.

### 5.2. ONNX Runtime oficial (em uso)

O ONNX Runtime da Microsoft, chamado pelo binding `github.com/yalue/onnxruntime_go`. É o runtime de produção pelo desempenho (§5.4) e por ser a implementação de referência do formato.

| Aspecto | Como fica |
|---------|-----------|
| Lib na VM | Pacote `onnxruntime` do repositório *community* do Alpine (`apk add onnxruntime`), disponível a partir do 3.23, em x86_64 e aarch64. No 3.24, ORT 1.24.4, ≈ 60 dependências e ≈ 58 MB em disco. A build glibc distribuída pela Microsoft não carrega na musl, nem com `gcompat` (faltam símbolos como `__snprintf_chk`); a lib precisa ser a do Alpine |
| Build | `CGO_ENABLED=1`, com `gcc` e `musl-dev` ([Build e deploy §3](03_build_e_deploy.md#3-bot-go)). O binário é dinâmico, ligado à musl, e abre a lib por `dlopen` no caminho `modelo.onnxruntime_lib` ([Configuração §1](11_configuracao.md#1-bot)) |
| Versão do binding | Fixada pela versão da lib: cada versão do binding exige uma API mínima do ORT. Para o ORT 1.24 do Alpine 3.24, `onnxruntime_go` **v1.27.0** (API 24); v1.30.0 já pede a 25, v1.37.0 a 29 e v1.38.0 a 31. Atualizar o Alpine e o binding é um passo único: sobe-se a versão do Alpine, o binding vai para a maior versão cuja API a nova lib suporta, e o teste de conformidade roda antes do deploy |
| Sessão | Criada a partir dos bytes do pacote (`NewDynamicAdvancedSessionWithONNXData`), com 1 thread intra-op e 1 inter-op: a VM `bot` tem 1 OCPU e as inferências são pequenas e esparsas |

Validado em experimento local, no `alpine:3.24` estável e no `alpine:edge` (ORT 1.30.0 com binding v1.37.0), com diferença máxima de 6e-8 em relação ao PyTorch. O ORT executa também grafos fora do contrato de §5.1 (opset 17, número de frases dinâmico), mas o pacote não os usa, para manter o gonnx como alternativa.

### 5.3. gonnx (alternativa)

`github.com/advancedclimatesystems/gonnx` v1.1.0, Go puro. Com `-tags gonnx`, o binário volta a ser estático (`CGO_ENABLED=0`, 12,9 MB) e a VM dispensa o pacote `onnxruntime`. Serve para o caso de o ORT ficar indisponível no Alpine ou de o acoplamento de versões de §5.2 travar uma atualização. Validado em experimento local num `alpine:3.20` sem nenhuma biblioteca adicional, com diferença máxima de 6e-8 em relação ao PyTorch, para 1, 3, 6 e 8 frases válidas.

### 5.4. Desempenho medido

Mesmo arquivo ONNX (o contrato de §5.1, K = 12), `alpine:3.24` limitado a 1 CPU, 200 execuções de aquecimento e 5 000 medidas:

| | ONNX Runtime (CGO) | gonnx |
|---|---|---|
| Média | **56 µs** | 643 µs |
| p50 | **54 µs** | 245 µs |
| p99 | **115 µs** | 1,41 ms |
| Pior caso | **0,2 ms** | 57,7 ms |
| Carga do modelo | 5,7 ms | 1,8 ms |
| Memória do processo (pico) | 32 MB | 22 MB |
| Binário | 2,1 MB + lib do sistema (≈ 58 MB) | 12,9 MB, sem dependências |

As duas opções ficam muito abaixo do tempo de qualquer resposta pelo WhatsApp; a diferença que pesa é a cauda — o gonnx aloca muito por inferência e sofre pausas do coletor de lixo — além da estabilidade de usar a implementação de referência.

Próximo: [Bot e modos de grupo →](09_bot_grupos.md)

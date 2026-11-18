# 07 — Pacote do modelo

Índice: [← Índice](README.md) | [← Treinamento](06_treinamento.md) | [Próximo: Carga do modelo →](08_carga_do_modelo.md)

Este documento é o dono do formato do arquivo `.bin` que o treino publica e o bot carrega: o esquema Protocol Buffers, o significado de cada campo, o versionamento e o nome do objeto no bucket. Como o pacote é gerado está em [Treinamento §7](06_treinamento.md#7-publicação); como é carregado, em [Carga do modelo](08_carga_do_modelo.md).

## 1. Esquema

`proto/modelo.proto`, compilado para Go e Python no build ([Build e deploy §2](03_build_e_deploy.md#2-código-gerado-do-protocol-buffers)):

```protobuf
syntax = "proto3";

package microfaqai.modelo.v1;

option go_package = "github.com/allan-rcos/micro-faq-ai/bot/internal/modelopb";

message Pacote {
  Versao                versao              = 1;
  uint32                versao_normalizacao = 2;
  bytes                 onnx                = 3;
  MiniDicionario        dicionario          = 4;
  repeated string       topicos             = 5;  // índice da saída do modelo → topic_id
  repeated Substituicao substituicoes       = 6;
  Metricas              metricas            = 7;
  bytes                 sha256_treinamento  = 8;  // SHA-256 do JSON de treinamento usado
}

message Versao {
  uint32 major        = 1;
  uint32 minor        = 2;
  uint32 patch        = 3;
  int64  gerado_em_ms = 4;  // instante UTC do fim do treino, em milissegundos Unix
  string hash         = 5;  // 8 primeiros hex do SHA-256 de gerado_em (ISO 8601)
}

message MiniDicionario {
  uint32          dimensao = 1;  // 300
  repeated string palavras = 2;  // palavras[i] ↔ linha i de vetores; 0 = <pad>, 1 = <unk>
  bytes           vetores  = 3;  // float32 little-endian, linha a linha: len(palavras) × dimensao
}

message Substituicao {
  string de   = 1;  // chave já normalizada
  string para = 2;  // valor já normalizado
}

message Metricas {
  uint32 exemplos           = 1;
  uint32 exemplos_validacao = 2;
  float  acuracia_validacao = 3;
  float  f1_macro_validacao = 4;
  float  temperatura        = 5;
}
```

## 2. Campos

| Campo | Para que o bot usa |
|-------|--------------------|
| `versao` | Escolher o pacote mais recente e registrar qual está em uso (§3) |
| `versao_normalizacao` | Recusar pacotes treinados com um pipeline de normalização que o binário não implementa ([Normalização §6](04_normalizacao.md#6-paridade-entre-go-e-python)) |
| `onnx` | O modelo serializado; entradas e saídas em [Treinamento §6](06_treinamento.md#6-exportação-onnx) |
| `dicionario` | O mini-dicionário: converter tokens em vetores ([Treinamento §3](06_treinamento.md#3-poda-para-o-mini-dicionário)) |
| `topicos` | O mapa de tópicos: a posição *i* de `probabilidades` é o tópico `topicos[i]`. Inclui os tópicos reservados ([Dados §1.2](05_dados.md#12-tópicos-reservados)) |
| `substituicoes` | Construir o autômato de Aho-Corasick ([Normalização §4](04_normalizacao.md#4-substituição-de-gírias-aho-corasick)); é a cópia com que o modelo foi treinado, já normalizada, então o bot não lê substituições do JSON |
| `metricas` | Registro em log e na mensagem de troca de modelo ao grupo de administração |
| `sha256_treinamento` | Rastrear de qual versão do JSON o pacote saiu |

`substituicoes` é um campo `repeated` porque `map` em Protocol Buffers não tem ordem de serialização garantida, e o pacote precisa ser determinístico; a ordem das substituições não afeta o casamento.

## 3. Versionamento

A versão tem duas partes com papéis distintos:

- **`major.minor.patch`** é a versão do **script de treino**, uma constante no código Python:
  - `major` muda quando o pacote deixa de ser compatível com o bot (campos, entradas do ONNX, semântica da representação vetorial). O bot só carrega o `major` que implementa.
  - `minor` muda com alterações compatíveis (arquitetura interna, hiperparâmetros).
  - `patch` muda com correções.
- **`gerado_em_ms` + `hash`** identificam a **carga**: dois treinos do mesmo script têm o mesmo semver e se distinguem por eles. `gerado_em_ms` dá a ordem; `hash` dá um identificador curto para logs e mensagens.

A forma textual é `1.2.0+20261117T203000Z.a1b2c3d4`. Entre pacotes de `major` compatível, o mais recente é o de maior `gerado_em_ms`.

## 4. Nome no bucket

```text
modelos/<AAAAMMDDTHHMMSSZ>_<major>.<minor>.<patch>_<hash>.bin
modelos/20261117T203000Z_1.2.0_a1b2c3d4.bin
```

O carimbo de tempo vem primeiro para que a ordem lexicográfica da listagem seja a ordem cronológica. O objeto leva o metadado `opc-meta-sha256` com o SHA-256 do arquivo inteiro, conferido pelo bot antes de desserializar.

## 5. Tamanho

| Parte | Ordem de grandeza |
|-------|-------------------|
| `dicionario.vetores` | 10 mil palavras × 300 × 4 bytes ≈ 12 MB |
| `onnx` | < 1 MB (≈ 77 mil parâmetros mais `K × 257`) |
| Demais campos | Alguns kB |

## 6. Retenção e backup

Todo pacote publicado é guardado para sempre fora do bucket, junto com o JSON de treinamento que o gerou; o bucket guarda só os mais recentes, para o bot e para um retorno rápido.

**No bucket.** Ficam os `publicacao.modelos_no_bucket` pacotes mais recentes (3 por padrão, [Configuração §2](11_configuracao.md#2-treino)). Com treino mensal, isso dá três meses de modelos à mão, qualquer que seja o intervalo configurado: a contagem é por versões, não por idade.

**Em releases do repositório.** A cada publicação, o treino cria uma release no repositório do projeto ([Treinamento §7](06_treinamento.md#7-publicação)):

| Item | Valor |
|------|-------|
| Tag | `modelo-<AAAAMMDDTHHMMSSZ>-<major>.<minor>.<patch>-<hash>`, a mesma identidade do nome no bucket (§4) |
| Anexos | `<nome do pacote>.bin.br` e `treinamento-<AAAAMMDDTHHMMSSZ>.json.br` |
| Descrição | Versão, número de tópicos, métricas e o SHA-256 dos dois arquivos sem compressão |

- A compressão é **Brotli** no nível máximo (qualidade 11, janela de 2²⁴ bytes). O JSON, texto repetitivo, encolhe bastante; o pacote encolhe pouco, porque é quase todo vetores `float32`, que se repetem pouco.
- As releases nunca são apagadas: são o histórico completo de modelos e dados.
- O acesso ao repositório fica atrás de uma interface do treino, `Releases`, com uma implementação por plataforma. A implementação do GitHub usa a API REST com um token de acesso restrito ao repositório (permissão de escrita em *contents*); GitLab ou Gitea/Forgejo entram como outras implementações da mesma interface, escolhidas por `releases.provedor`.

**Retorno a um modelo anterior.** Basta apagar do bucket os pacotes mais novos que ele: o bot carrega sempre o mais recente que restar ([Carga do modelo §1](08_carga_do_modelo.md#1-descoberta)). Um pacote que já saiu do bucket é baixado da release, descomprimido (`brotli -d`) e enviado de volta para `modelos/` com o mesmo nome.

Próximo: [Carga do modelo no bot →](08_carga_do_modelo.md)

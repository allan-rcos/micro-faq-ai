# 03 — Build e deploy

Índice: [← Índice](README.md) | [← Infraestrutura](02_infraestrutura.md) | [Próximo: Normalização →](04_normalizacao.md)

Este documento descreve como cada artefato é gerado, minimizado e instalado nas VMs. Os formatos internos dos artefatos ficam com os seus donos: o pacote do modelo em [Pacote do modelo](07_pacote_modelo.md) e o dicionário estático em [Treinamento §2](06_treinamento.md#2-dicionário-estático).

## 1. Princípio

Nenhuma VM compila, baixa dependência ou instala pacote de linguagem. Todo artefato é gerado na máquina de desenvolvimento (ou em CI), para a arquitetura da VM de destino — `linux/arm64` para o bot, `linux/amd64` para o treino ([Infraestrutura §2](02_infraestrutura.md#2-máquinas-virtuais)) —, já minimizado, e copiado pronto para o servidor.

| Artefato | Gerado por | Destino | Tamanho esperado |
|----------|-----------|---------|------------------|
| `micro-faq-bot` (binário Go) | `go build` com CGO, ligado à musl (§3) | VM `bot`, `/usr/local/bin/` | 15–25 MB (5–8 MB com UPX) |
| `treino.tar.zst` (Python + dependências + scripts) | Contêiner Debian slim amd64 (§4) | VM `treino`, `/opt/micro-faq-ai/` | ≈ 250 MB descompactado |
| Dicionário estático convertido | Script de conversão (§5) | VM `treino`, `/var/lib/micro-faq-ai/dicionario/` | ≈ 2,5 GB |

## 2. Código gerado do Protocol Buffers

O esquema `proto/modelo.proto` ([Pacote do modelo §1](07_pacote_modelo.md#1-esquema)) é compilado antes dos dois builds:

```bash
protoc --go_out=bot/internal/modelopb --go_opt=paths=source_relative \
       --python_out=treino/modelopb proto/modelo.proto
```

O código gerado entra no build de cada lado; o `.proto` é a única definição do formato.

## 3. Bot (Go)

O build roda num contêiner `golang:1.24-alpine` para `linux/arm64` — nativo numa máquina arm64, ou com `--platform linux/arm64` sob QEMU numa x86 —, para que o binário seja ligado à mesma musl da VM:

```bash
apk add gcc musl-dev
CGO_ENABLED=1 GOOS=linux GOARCH=arm64 \
go build -trimpath -tags "ort timetzdata" \
  -ldflags="-s -w -X main.versao=$(git describe --tags)" \
  -o dist/micro-faq-bot ./bot/cmd/micro-faq-bot
upx --best --lzma dist/micro-faq-bot   # opcional
```

- O CGO existe só por causa do ONNX Runtime ([Carga do modelo §5.2](08_carga_do_modelo.md#52-onnx-runtime-oficial-em-uso)); todas as outras dependências são Go puro. Trocando `-tags "ort timetzdata"` por `-tags "gonnx timetzdata"` e `CGO_ENABLED=0`, o mesmo código gera um binário estático com o runtime alternativo ([Carga do modelo §5.3](08_carga_do_modelo.md#53-gonnx-alternativa)).

| Função | Dependência |
|--------|-------------|
| WhatsApp | `go.mau.fi/whatsmeow` |
| Object Storage | `github.com/oracle/oci-go-sdk/v65` |
| NoSQL | `github.com/oracle/nosql-go-sdk` |
| Protocol Buffers | `google.golang.org/protobuf` |
| Configuração TOML | `github.com/BurntSushi/toml` |
| Identificadores de tópico | nanoID (`github.com/matoous/go-nanoid/v2`) |
| Normalização Unicode | `golang.org/x/text/unicode/norm` |
| Aho-Corasick | implementação Go pura ([Normalização §4](04_normalizacao.md#4-substituição-de-gírias-aho-corasick)) |
| Inferência ONNX | `github.com/yalue/onnxruntime_go` v1.27.0, com CGO, ou `github.com/advancedclimatesystems/gonnx` v1.1.0, Go puro — um ou outro por *build tag*, atrás da interface de [Carga do modelo §5.1](08_carga_do_modelo.md#51-runtime-onnx) |

- `-s -w` remove tabela de símbolos e DWARF; `-trimpath` remove caminhos locais; `-tags timetzdata` embute o banco de fusos horários.
- UPX reduz o arquivo em disco ao custo de descompactar na memória ao iniciar; com 12 GB de RAM o custo é irrelevante.
- Na VM, os pacotes de sistema necessários são `ca-certificates` e `onnxruntime` (`apk add ca-certificates onnxruntime`). A versão do `onnxruntime` vem com a do Alpine e fixa a do binding ([Carga do modelo §5.2](08_carga_do_modelo.md#52-onnx-runtime-oficial-em-uso)).

## 4. Treino (Python)

Python não vira binário nativo com PyTorch sem perder suporte; o que se minimiza é o ambiente. O build roda num contêiner `debian:12-slim` `linux/amd64`, com a mesma versão de Python da VM (3.11, que já traz `tomllib` para ler a configuração), para que as extensões nativas sejam compatíveis:

```bash
python3 -m venv /opt/micro-faq-ai/venv
/opt/micro-faq-ai/venv/bin/pip install --no-cache-dir \
  --index-url https://download.pytorch.org/whl/cpu torch
/opt/micro-faq-ai/venv/bin/pip install --no-cache-dir \
  numpy onnx onnxruntime oci protobuf brotli
```

Depois da instalação, o ambiente é podado:

1. `torch` **CPU-only** (o índice `whl/cpu`), sem bibliotecas CUDA.
2. Remoção de `tests/`, `include/`, `*.a`, `*.pyi`, `__pycache__/` e documentação dos pacotes; `strip --strip-unneeded` em todos os `*.so`.
3. Os scripts do projeto (`treino/`) são compilados com `python -OO -m compileall -b` e os `.py` originais são removidos; as dependências mantêm os fontes, porque o exportador ONNX do PyTorch inspeciona código-fonte.
4. Tudo é empacotado em `treino.tar.zst`.

Na VM, os pacotes de sistema são `python3` (a mesma versão do contêiner), `cron` e `zstd`, e um arquivo de swap de 4 GB ([Treinamento §1.1](06_treinamento.md#11-recursos-da-vm)). O Aho-Corasick e a normalização são código do projeto ([Normalização §1](04_normalizacao.md#1-etapas)), sem biblioteca.

## 5. Dicionário estático

O arquivo original do fastText (`cc.pt.300.bin`, ≈ 7 GB) é convertido uma vez, fora do servidor, para matrizes `float16` lidas por *memory map* e com o vocabulário já normalizado. O formato e a razão de cada parte estão em [Treinamento §2](06_treinamento.md#2-dicionário-estático). A saída é copiada para a VM `treino` e também para `estatico/` no bucket ([Dados §3](05_dados.md#3-layout-do-bucket)), para recriar a VM sem refazer a conversão.

## 6. Instalação nas VMs

**Bot (Alpine):**

```bash
scp dist/micro-faq-bot bot:/usr/local/bin/micro-faq-bot.novo
ssh bot 'mv /usr/local/bin/micro-faq-bot /usr/local/bin/micro-faq-bot.anterior;
         mv /usr/local/bin/micro-faq-bot.novo /usr/local/bin/micro-faq-bot;
         rc-service micro-faq-bot restart'
```

Na primeira instalação, `micro-faq-bot preparar` cria a tabela, os tópicos reservados e o JSON inicial ([Ambiente local §3](12_ambiente_local.md#3-subindo)); é o mesmo comando do ambiente local e não altera nada que já exista.

O serviço OpenRC `/etc/init.d/micro-faq-bot` roda o binário como usuário `microfaq`, com `/etc/micro-faq-ai/bot.toml` ([Configuração §1](11_configuracao.md#1-bot)). Reverter é trocar de volta pelo `.anterior`.

**Treino (Debian):**

```bash
scp dist/treino.tar.zst treino:/tmp/
ssh treino 'tar --zstd -xf /tmp/treino.tar.zst -C /opt/micro-faq-ai'
```

O cron que dispara o treinamento está em [Configuração §2](11_configuracao.md#2-treino).

Próximo: [Normalização de texto →](04_normalizacao.md)

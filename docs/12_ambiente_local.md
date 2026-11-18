# 12 — Ambiente local

Índice: [← Índice](README.md) | [← Configuração](11_configuracao.md) | [Próximo: Questões em aberto →](13_questoes_em_aberto.md)

Este documento descreve como subir o sistema inteiro na máquina do desenvolvedor com Docker: bot, treino e banco, sem nenhuma conta na OCI. O código é o mesmo da produção; o que muda é a configuração ([Configuração](11_configuracao.md)), que escolhe implementações locais para as dependências externas.

## 1. Serviços

Arquivo `deploy/docker/compose.yaml`:

| Serviço | Imagem | Papel |
|---------|--------|-------|
| `nosql` | `ghcr.io/oracle/nosql:latest-ce` | Oracle NoSQL Community Edition com o proxy HTTP na porta 8080. Aceita a mesma tabela, as mesmas consultas e o TTL da versão na nuvem, pelo mesmo SDK |
| `bot` | `deploy/docker/bot.Dockerfile`: estágio de build com os comandos de [Build e deploy §3](03_build_e_deploy.md#3-bot-go), estágio final `alpine:3.24` + `onnxruntime` | O bot, com `deploy/docker/bot.toml` |
| `treino` | `deploy/docker/treino.Dockerfile`: `debian:12-slim` com o ambiente de [Build e deploy §4](03_build_e_deploy.md#4-treino-python) | O treino, rodado sob demanda em vez de pelo cron, com `deploy/docker/treino.toml` |

| Volume | Conteúdo |
|--------|----------|
| `bucket` | O papel do bucket: `dados/`, `modelos/`, com o mesmo layout de [Dados §3](05_dados.md#3-layout-do-bucket) |
| `dicionario` | O dicionário estático convertido (§4) |

As imagens são construídas para a arquitetura da máquina do desenvolvedor. Os artefatos de produção continuam sendo gerados como em [Build e deploy](03_build_e_deploy.md), para a arquitetura de cada VM.

## 2. Implementações locais

Cada dependência externa fica atrás de uma interface, e a configuração escolhe a implementação na inicialização. O código de negócio nunca pergunta se está rodando localmente.

| Dependência | Interface | Produção | Local |
|-------------|-----------|----------|-------|
| Bucket | `Armazenamento`, no bot e no treino | OCI Object Storage (`armazenamento.tipo = "oci"`) | Diretório no volume `bucket` (`tipo = "local"`). O `ETag` é o SHA-256 do conteúdo, e a gravação condicional ([Dados §2.2](05_dados.md#22-gravação-concorrente)) usa um arquivo de trava e renomeação atômica |
| NoSQL | O próprio SDK `nosql-go-sdk` | Nuvem, com autenticação da OCI (`nosql.modo = "nuvem"`) | Serviço `nosql`, em modo *on-premises* sem segurança (`modo = "local"`, `endpoint`) |
| Releases | `Releases`, no treino | GitHub ([Pacote do modelo §6](07_pacote_modelo.md#6-retenção-e-backup)) | Desligado (`releases.ativo = false`), ou apontado para um repositório de testes |
| Runtime ONNX | `inferencia` ([Carga do modelo §5.1](08_carga_do_modelo.md#51-runtime-onnx)) | ONNX Runtime | O mesmo, do pacote do Alpine na imagem |
| WhatsApp | — | Número dedicado | Um número de testes, vinculado da mesma forma ([Bot §1](09_bot_grupos.md#1-conexão-com-o-whatsapp)) |

## 3. Subindo

```bash
cd deploy/docker
docker compose up -d nosql
docker compose run --rm bot micro-faq-bot preparar   # tabela, tópicos reservados e JSON inicial
docker compose run --rm bot micro-faq-bot vincular   # QR code do número de testes
docker compose up -d bot
docker compose run --rm treino treinar --forcar    # um treino agora, sem esperar o intervalo
```

- `micro-faq-bot preparar` é o mesmo comando usado na primeira instalação em produção: cria a tabela `micro_faq` se ela não existe, grava os tópicos reservados que faltam ([Dados §1.2](05_dados.md#12-tópicos-reservados)) e cria `dados/treinamento.json` com os exemplos de partida se ele não existe. Rodar de novo não altera nada que já exista.
- `treinar --forcar` ignora as duas condições do disparo ([Treinamento §1](06_treinamento.md#1-disparo)) e treina com o JSON atual.
- O bot do ambiente local encontra o pacote novo no volume `bucket` pela descoberta normal ([Carga do modelo §1](08_carga_do_modelo.md#1-descoberta)).

## 4. Dicionário reduzido

O dicionário estático completo ocupa ≈ 2,5 GB. Para desenvolvimento, o script de conversão ([Build e deploy §5](03_build_e_deploy.md#5-dicionário-estático)) aceita `--limite 200000`, que mantém só as 200 mil palavras mais frequentes e todos os *buckets* de n-gramas. O formato é o mesmo de [Treinamento §2.1](06_treinamento.md#21-formato-convertido); muda só o tamanho de `vetores.f16.npy`, e palavras raras passam a receber o vetor por n-gramas.

Próximo: [Questões em aberto →](13_questoes_em_aberto.md)

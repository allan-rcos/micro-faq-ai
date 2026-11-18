# 11 — Configuração

Índice: [← Índice](README.md) | [← Fluxo de resposta](10_fluxo_de_resposta.md) | [Próximo: Ambiente local →](12_ambiente_local.md)

Referência dos arquivos de configuração das duas VMs, com os valores padrão. O significado de cada chave está no documento linkado na tabela; aqui ficam só os nomes e os padrões. As duas configurações são arquivos **TOML** — lidos com `github.com/BurntSushi/toml` no Go e com `tomllib`, da biblioteca padrão, no Python. Fica em TOML o que é escolhido por quem instala; o que muda com o uso (tópicos, casos, sessão do WhatsApp) fica na tabela única ([Dados §1](05_dados.md#1-tabela-única-nosql)).

## 1. Bot

Arquivo `/etc/micro-faq-ai/bot.toml`:

```toml
[oci]
regiao        = "sa-saopaulo-1"
compartimento = "ocid1.compartment.oc1..xxxx"
autenticacao  = "instancia"        # ou "arquivo"
arquivo       = "~/.oci/config"    # só no modo "arquivo"
perfil        = "DEFAULT"

[armazenamento]
tipo      = "oci"                  # ou "local"
namespace = "xxxx"
bucket    = "micro-faq-ai"
diretorio = "/dados/bucket"        # só no tipo "local"

[nosql]
modo     = "nuvem"                 # ou "local"
tabela   = "micro_faq"
endpoint = "http://nosql:8080"     # só no modo "local"

[[grupos]]
jid  = "120363000000000001@g.us"
nome = "Turma ADS 2026"
modo = "marcacao"

[[grupos]]
jid  = "120363000000000002@g.us"
nome = "Avisos e dúvidas"
modo = "autonomo"

[[grupos]]
jid  = "120363000000000003@g.us"
nome = "Admin bot"
modo = "administracao"

[faixas]
media = 50
alta  = 80

[alternativas]
maximo               = 3
probabilidade_minima = 10
expira_em            = "10m"

[negativa]
janela = "5m"

[cumprimento]
intervalo = "30m"

[autonomo]
espera_digitando = "8s"
espera_maxima    = "45s"

[autonomo.inundacao]
mensagens = 30
janela    = "60s"

[modelo]
verificar_a_cada    = "5m"
teste_acerto_minimo = 90
onnxruntime_lib     = "/usr/lib/libonnxruntime.so.1"

[casos]
expira_em = "72h"
```

| Chave | Padrão | Definida em |
|-------|--------|-------------|
| `oci.*` | `autenticacao = "instancia"`; os demais obrigatórios | [Infraestrutura §5](02_infraestrutura.md#5-identidade-e-permissões) |
| `armazenamento.*` | `tipo = "oci"` | [Dados §3](05_dados.md#3-layout-do-bucket); o tipo `local` em [Ambiente local §2](12_ambiente_local.md#2-implementações-locais) |
| `nosql.*` | `modo = "nuvem"`, `tabela = "micro_faq"` | [Dados §1](05_dados.md#1-tabela-única-nosql); o modo `local` em [Ambiente local §2](12_ambiente_local.md#2-implementações-locais) |
| `grupos[].modo` | — (obrigatório) | [Bot §2](09_bot_grupos.md#2-modos-de-grupo) |
| `faixas.media`, `faixas.alta` | `50`, `80` | [Fluxo §1](10_fluxo_de_resposta.md#1-decisão) |
| `alternativas.*` | `3`, `10`, `10m` | [Fluxo §4](10_fluxo_de_resposta.md#4-confiança-média-alternativas) |
| `negativa.janela` | `5m` | [Fluxo §5](10_fluxo_de_resposta.md#5-resposta-negativa) |
| `cumprimento.intervalo` | `30m` | [Fluxo §2](10_fluxo_de_resposta.md#2-tópicos-reservados) |
| `autonomo.espera_digitando`, `autonomo.espera_maxima` | `8s`, `45s` | [Bot §2.2](09_bot_grupos.md#22-autônomo) |
| `autonomo.inundacao.*` | `30`, `60s` | [Bot §2.2](09_bot_grupos.md#22-autônomo) |
| `modelo.verificar_a_cada` | `5m` | [Carga do modelo §1](08_carga_do_modelo.md#1-descoberta) |
| `modelo.teste_acerto_minimo` | `90` | [Carga do modelo §3](08_carga_do_modelo.md#3-teste) |
| `modelo.onnxruntime_lib` | `/usr/lib/libonnxruntime.so.1` | [Carga do modelo §5.2](08_carga_do_modelo.md#52-onnx-runtime-oficial-em-uso); ignorada no build com gonnx |
| `casos.expira_em` | `72h` | [Fluxo §6.1](10_fluxo_de_resposta.md#61-caso); vira o TTL da linha do caso, em horas inteiras ([Dados §1.3](05_dados.md#13-caso)) |

Validação na inicialização — qualquer falha encerra o processo com erro:

- exatamente um grupo com `modo = "administracao"`;
- nenhum `jid` repetido;
- `0 < faixas.media < faixas.alta ≤ 100`;
- `casos.expira_em` de pelo menos 1 hora.

## 2. Treino

Arquivo `/etc/micro-faq-ai/treino.toml`:

```toml
[oci]
regiao       = "sa-saopaulo-1"
autenticacao = "instancia"         # ou "arquivo"
arquivo      = "~/.oci/config"
perfil       = "DEFAULT"

[armazenamento]
tipo      = "oci"                  # ou "local"
namespace = "xxxx"
bucket    = "micro-faq-ai"
diretorio = "/dados/bucket"

[agenda]
intervalo_dias = 30

[publicacao]
modelos_no_bucket = 3

[releases]
ativo         = true
provedor      = "github"           # ou "gitlab", "gitea"
repositorio   = "allan-rcos/micro-faq-ai"
token_arquivo = "/etc/micro-faq-ai/releases.token"

[dicionario]
diretorio = "/var/lib/micro-faq-ai/dicionario"

[modelo]
oculta           = 256
dropout          = 0.3
descarte_tokens  = 0.1
taxa_aprendizado = 0.001
weight_decay     = 0.0001
tamanho_lote     = 32
epocas_max       = 200
paciencia        = 15
validacao        = 0.15
semente          = 42
f1_minimo        = 0.7
```

| Chave | Padrão | Definida em |
|-------|--------|-------------|
| `oci.*`, `armazenamento.*` | Como no bot | [Infraestrutura §5](02_infraestrutura.md#5-identidade-e-permissões), [Ambiente local §2](12_ambiente_local.md#2-implementações-locais) |
| `agenda.intervalo_dias` | `30` | [Treinamento §1](06_treinamento.md#1-disparo) |
| `publicacao.modelos_no_bucket` | `3` | [Pacote do modelo §6](07_pacote_modelo.md#6-retenção-e-backup) |
| `releases.*` | `ativo = true` | [Pacote do modelo §6](07_pacote_modelo.md#6-retenção-e-backup); o arquivo do token tem permissão `600` |
| `dicionario.diretorio` | `/var/lib/micro-faq-ai/dicionario` | [Treinamento §2](06_treinamento.md#2-dicionário-estático) |
| `modelo.*` | Os valores acima | [Treinamento §5](06_treinamento.md#5-modelo) |

O limite de 8 frases não é configurável, porque faz parte da forma de entrada do ONNX ([Treinamento §6](06_treinamento.md#6-exportação-onnx)).

Crontab do usuário `microfaq` (`crontab -e`), uma verificação por dia às 3h:

```cron
0 3 * * * flock -n /run/lock/micro-faq-treino.lock nice -n 19 /opt/micro-faq-ai/bin/treinar >> /var/log/micro-faq-ai/treino.log 2>&1
```

A verificação diária é barata; o treino de fato só acontece a cada `agenda.intervalo_dias` e quando há dados novos ([Treinamento §1](06_treinamento.md#1-disparo)).

Próximo: [Ambiente local →](12_ambiente_local.md)

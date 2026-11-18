# 05 — Dados

Índice: [← Índice](README.md) | [← Normalização](04_normalizacao.md) | [Próximo: Treinamento →](06_treinamento.md)

Este documento é o dono dos três repositórios de dados persistentes: a tabela única no NoSQL (tópicos, casos e sessão do WhatsApp), o JSON de treinamento e o layout do bucket. Os recursos da OCI que os hospedam estão em [Infraestrutura](02_infraestrutura.md); o pacote do modelo, que também fica no bucket, tem formato próprio em [Pacote do modelo](07_pacote_modelo.md).

## 1. Tabela única (NoSQL)

Tudo que o bot guarda no NoSQL fica numa só tabela, em *single table design*, com chave de partição (`pk`) e chave de ordenação (`sk`) em todos os itens. O conteúdo é um documento JSON.

```sql
CREATE TABLE IF NOT EXISTS micro_faq (
  pk    STRING,
  sk    STRING,
  dados JSON,
  PRIMARY KEY (SHARD(pk), sk)
)
```

| Item | `pk` | `sk` | `dados` | TTL |
|------|------|------|---------|-----|
| Tópico (§1.1) | `topico` | `<topic_id>` | `{"resposta": "...", "exemplo": "..."}` | Sem expiração |
| Caso (§1.3) | `caso` | `<id da mensagem do caso>` | Ver §1.3 | `casos.expira_em` ([Configuração §1](11_configuracao.md#1-bot)) |
| Sessão do WhatsApp (§1.4) | `wa#<JID do dispositivo>` | `<tipo>#<chave>` | Ver §1.4 | Sem expiração |

- **Padrão de chave.** `pk` agrupa os itens de um mesmo tipo (ou de um mesmo dono) e `sk` identifica o item dentro do grupo. O NoSQL exige valor em todas as colunas da chave, então um item que não precisa de `sk` a grava vazia (`""`).
- **Leituras.** Quase todas são `Get` por `pk` + `sk`. As únicas consultas por grupo são de uma partição só: `SELECT * FROM micro_faq WHERE pk = "topico"`, para o comando `/topicos` ([Fluxo §6.2](10_fluxo_de_resposta.md#62-comandos)) e para a verificação de um pacote novo ([Carga do modelo §2](08_carga_do_modelo.md#2-verificação)).
- **Sem cache.** O bot lê o item a cada uso. Uma leitura por chave custa pouco, e a memória do bot fica para o mini-dicionário e o modelo.
- Nenhuma análise é feita sobre a tabela.

### 1.1. Tópico

| Campo | Conteúdo |
|-------|----------|
| `topic_id` (`sk`) | Identificador curto do tópico, um **nanoID** de 8 caracteres em `0-9a-z` (`k3f9q2x1`). É sempre gerado pelo bot, quando um administrador cria um tópico para uma dúvida que o modelo não identificou ([Fluxo §6.2](10_fluxo_de_resposta.md#62-comandos)); a gravação é condicional (só se a chave não existe), e uma colisão gera outro identificador |
| `resposta` | Texto enviado ao grupo quando a pergunta é classificada neste tópico, com a formatação do WhatsApp |
| `exemplo` | O **exemplo canônico**: uma única pergunta, escrita como um usuário escreveria, que representa o tópico. É mostrado nas alternativas de confiança média ([Fluxo §4](10_fluxo_de_resposta.md#4-confiança-média-alternativas)), ao lado de cada identificador nas mensagens aos administradores ([Fluxo §6.1](10_fluxo_de_resposta.md#61-caso)) e usado no teste de carga do modelo ([Carga do modelo §3](08_carga_do_modelo.md#3-teste)) |

Todo `topic_id` do mapa de tópicos de um pacote precisa existir na tabela; a carga do modelo verifica isso.

### 1.2. Tópicos reservados

Três tópicos têm identificador fixo, começado por `_` (que os nanoIDs nunca usam), e significado conhecido pelo bot. São treinados como qualquer outro tópico, com exemplos no JSON de treinamento, mas o bot não os trata como dúvidas: cada um dispara um comportamento próprio ([Fluxo §2](10_fluxo_de_resposta.md#2-tópicos-reservados)).

| `topic_id` | Reconhece | `resposta` |
|------------|-----------|------------|
| `_cumprimento` | "bom dia", "oi", "olá", "boa noite, pessoal" | O cumprimento genérico enviado de volta |
| `_negacao` | "não", "não é isso", "errado", "nada a ver" | Vazia; só tem sentido como resposta a uma resposta do bot |
| `_conversa` | Mensagens que não são dúvida: "kkkk", "valeu", "chegando aí" | Vazia; o bot não responde |

- Os três precisam existir na tabela e ter exemplos no JSON; o JSON inicial do projeto já os traz com exemplos de partida.
- O treino falha se algum deles estiver sem exemplos ([Treinamento §1](06_treinamento.md#1-disparo)), e a carga do modelo recusa um pacote sem os três ([Carga do modelo §2](08_carga_do_modelo.md#2-verificação)).
- O `_conversa` é o que permite ao bot ler todas as mensagens de um grupo autônomo sem mandar a conversa comum para os administradores.

### 1.3. Caso

Um caso é uma dúvida aberta no grupo de administração ([Fluxo §6.1](10_fluxo_de_resposta.md#61-caso)). A `sk` é o identificador (`StanzaID`) da mensagem do caso no grupo de administração, porque é o que o bot recebe quando um administrador responde citando essa mensagem.

```json
{
  "grupo": "120363000000000001@g.us",
  "mensagem": "3EB0C1A2B3C4D5E6F7",
  "autor": "5541999990000@s.whatsapp.net",
  "texto": "qnd sai o resultado da prova substitutiva?",
  "palpites": [["k3f9q2x1", 0.47], ["p8x1za0m", 0.21]],
  "aberto_em": "2026-11-17T20:30:00Z"
}
```

O TTL da linha encerra o caso sozinho quando ninguém responde; um caso resolvido é apagado.

### 1.4. Sessão do WhatsApp

O estado criptográfico e de sincronização do dispositivo vinculado — identidade, chaves de sessão, chaves de grupo, estado de sincronização, contatos — também fica na tabela, numa partição por dispositivo. O tipo do dado é o prefixo da `sk`:

| `sk` | Conteúdo |
|------|----------|
| `""` | O próprio dispositivo: JID, chaves de identidade e de registro |
| `identidade#<endereço>` | Chave de identidade de um contato |
| `sessao#<endereço>` | Sessão Signal com um contato |
| `prekey#<id>` | Pre-keys do dispositivo |
| `senderkey#<grupo>#<remetente>` | Chave de remetente de um grupo |
| `appstate#<nome>` e `appstatekey#<id>` | Estado e chaves de sincronização do aplicativo |
| `contato#<jid>`, `lid#<lid>` | Contatos e mapeamento entre número e LID |

Como esses itens são lidos e gravados está em [Bot §1.1](09_bot_grupos.md#11-sessão-na-tabela-única).

## 2. JSON de treinamento

Objeto `dados/treinamento.json` no bucket. É o conjunto de exemplos rotulados e o dicionário de substituições.

```json
{
  "versao_formato": 1,
  "topicos": {
    "k3f9q2x1": [
      "quando abre a matrícula?",
      "qnd começa a matricula do semestre"
    ],
    "p8x1za0m": [
      "onde vejo minhas notas"
    ],
    "_cumprimento": ["bom dia", "oi gente, tudo bem?", "boa noite pessoal"],
    "_negacao": ["não", "não é isso", "nada a ver"],
    "_conversa": ["kkkkk", "valeu!", "to chegando"]
  },
  "substituicoes": {
    "vc": "você",
    "qnd": "quando",
    "pq": "por que"
  }
}
```

### 2.1. Regras

- Os exemplos são gravados **crus**, como chegaram do WhatsApp (sem a menção ao bot). A normalização é aplicada no treino, então uma mudança no pipeline ([Normalização](04_normalizacao.md)) vale para todo o histórico no treinamento seguinte.
- Um exemplo não é acrescentado se a sua forma normalizada já existe no mesmo tópico.
- Chaves e valores de `substituicoes` são gravados na forma legível; o treino os normaliza ao montar o dicionário ([Normalização §4](04_normalizacao.md#4-substituição-de-gírias-aho-corasick)).
- Só o bot grava este objeto; o treino só lê.

### 2.2. Gravação concorrente

Dentro do bot, as gravações passam por uma única fila, uma de cada vez. Mesmo assim o objeto pode mudar por fora (uma edição manual, por exemplo), então cada gravação é uma leitura–alteração–escrita condicional:

1. `GetObject` devolve o conteúdo e o `ETag`.
2. O bot aplica a alteração (novo exemplo, novo tópico) sobre o conteúdo lido.
3. `PutObject` com `If-Match: <ETag>`.
4. Se a resposta é `412 Precondition Failed`, o objeto mudou depois da leitura: volta ao passo 1, até 5 tentativas. Esgotadas as tentativas, a alteração continua na fila e é tentada de novo no próximo ciclo.

O versionamento do bucket guarda cada versão anterior do objeto ([Infraestrutura §3](02_infraestrutura.md#3-object-storage)).

## 3. Layout do bucket

```text
micro-faq-ai/
├── dados/
│   └── treinamento.json                       §2
├── modelos/
│   └── <AAAAMMDDTHHMMSSZ>_<semver>_<hash>.bin pacotes do modelo
└── estatico/
    └── dicionario/                            dicionário estático convertido (cópia de recuperação)
```

O nome dos pacotes é definido em [Pacote do modelo §4](07_pacote_modelo.md#4-nome-no-bucket), e quantos ficam no bucket em [Pacote do modelo §6](07_pacote_modelo.md#6-retenção-e-backup); o conteúdo de `estatico/dicionario/` em [Treinamento §2](06_treinamento.md#2-dicionário-estático).

Próximo: [Treinamento →](06_treinamento.md)

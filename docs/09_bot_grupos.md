# 09 — Bot e modos de grupo

Índice: [← Índice](README.md) | [← Carga do modelo](08_carga_do_modelo.md) | [Próximo: Fluxo de resposta →](10_fluxo_de_resposta.md)

Este documento descreve a conexão do bot com o WhatsApp e o que ele captura em cada modo de grupo: de que mensagens sai uma pergunta e quando ela está pronta para classificar. O que acontece com a pergunta depois de classificada é de [Fluxo de resposta](10_fluxo_de_resposta.md).

## 1. Conexão com o WhatsApp

- Biblioteca `go.mau.fi/whatsmeow`, conectada como **dispositivo vinculado** de um número dedicado ao bot.
- A sessão do dispositivo fica na tabela única do NoSQL (§1.1), então a VM não guarda estado: o bot pode ser recriado em outra máquina sem vincular o número de novo.
- Na primeira execução, `micro-faq-bot vincular` imprime o QR code no terminal (ou um código de pareamento por número, `PairPhone`) e encerra depois de vincular.
- Eventos tratados:

| Evento | Uso |
|--------|-----|
| `*events.Message` | Toda mensagem de grupo configurado |
| `*events.ChatPresence` | Estado *digitando* / *parou* de cada participante, no modo autônomo |
| `*events.Connected` | Envia `PresenceAvailable`, necessário para receber presença de grupos |
| `*events.LoggedOut` | Encerra o processo com erro; o OpenRC registra e não reinicia em laço |

- Mensagens de grupos fora da configuração e mensagens privadas são ignoradas.
- O bot identifica a si mesmo tanto pelo JID do número quanto pelo LID (`client.Store.ID` e `client.Store.LID`), porque menções em grupos podem vir em qualquer dos dois.

### 1.1. Sessão na tabela única

O whatsmeow guarda o estado do dispositivo através de um conjunto de interfaces (`store.IdentityStore`, `store.SessionStore`, `store.PreKeyStore`, `store.SenderKeyStore`, `store.AppStateStore`, `store.ContactStore`, `store.LIDStore` e as demais de `go.mau.fi/whatsmeow/store`) e de um contêiner de dispositivos. O pacote `bot/internal/sessaowa` implementa essas interfaces sobre a tabela única, com os itens de [Dados §1.4](05_dados.md#14-sessão-do-whatsapp).

- **Gravação síncrona.** Cada alteração é gravada antes de a biblioteca seguir. As chaves de sessão avançam a cada mensagem, e uma gravação perdida faz as mensagens seguintes daquele contato ou grupo não serem decifradas.
- **Sem cache.** Como no resto da tabela ([Dados §1](05_dados.md#1-tabela-única-nosql)), cada leitura vai ao NoSQL. Uma mensagem de grupo custa algumas leituras e gravações por chave.
- **Versão fixada.** As interfaces de `store` mudam entre versões do whatsmeow. A versão é fixada no `go.mod`, e o pacote `sessaowa` tem um teste que roda o ciclo de vínculo, envio e recebimento contra o NoSQL local ([Ambiente local](12_ambiente_local.md)) antes de cada atualização da biblioteca.
- **Desacoplamento.** Só `sessaowa` conhece o formato dos itens; o resto do bot recebe um `*store.Device` pronto, como receberia de qualquer outro armazenamento do whatsmeow.

## 2. Modos de grupo

Cada grupo configurado tem exatamente um modo ([Configuração §1](11_configuracao.md#1-bot)). Há exatamente um grupo de administração.

| Modo | Captura | Pronta para classificar |
|------|---------|-------------------------|
| `marcacao` | Só mensagens que mencionam o bot | Imediatamente |
| `autonomo` | Toda mensagem com texto | Quando o autor para de digitar |
| `administracao` | Nenhuma pergunta; recebe as dúvidas não resolvidas e os comandos dos administradores | — |

Em todos os modos, o texto de uma mensagem é o campo de texto (`Conversation` ou `ExtendedTextMessage.Text`) ou a legenda de imagem/vídeo/documento. Mensagens sem texto (áudio, figurinha) são ignoradas.

### 2.1. Marcação

Uma mensagem é capturada quando `ContextInfo.MentionedJID` contém o bot.

- **Menção em resposta a outra mensagem** (`ContextInfo.QuotedMessage` presente): a pergunta é a **mensagem citada**, e o autor da pergunta é o autor da citada (`ContextInfo.Participant`). É o caso de alguém marcar o bot numa dúvida de outra pessoa.
- **Menção sem citação**: a pergunta é a própria mensagem, sem o trecho `@número` da menção.
- Se a menção é tudo que a mensagem tem e não há citação, nada é capturado.

A resposta sempre cita a mensagem da pergunta, não a da menção.

### 2.2. Autônomo

O bot lê todas as mensagens do grupo e monta perguntas por **rajada**: as mensagens seguidas de um mesmo autor formam uma só pergunta.

1. **Abertura**: qualquer mensagem com texto abre uma rajada, ou entra na rajada aberta do mesmo autor. Cumprimentos e conversa comum não são filtrados aqui: o modelo os reconhece como tópicos reservados ([Fluxo §2](10_fluxo_de_resposta.md#2-tópicos-reservados)).
2. **Espera**: a rajada de cada `(grupo, autor)` fecha quando, depois da última mensagem, o autor fica `autonomo.espera_digitando` sem enviar nada e sem presença *digitando*. Um evento *digitando* adia o fechamento; um evento *parou* começa a contar a espera de novo. Para que um *digitando* perdido não segure a rajada para sempre, ela fecha de qualquer forma após `autonomo.espera_maxima` contada da primeira mensagem.
3. **Pergunta**: as mensagens da rajada, na ordem, viram uma só pergunta de várias frases ([Normalização §5](04_normalizacao.md#5-frases-e-tokens)). A resposta cita a última mensagem da rajada.

**Inundação.** O bot conta as mensagens de cada grupo autônomo numa janela deslizante. Se passam de `autonomo.inundacao.mensagens` em `autonomo.inundacao.janela`:

1. chama `SetGroupAnnounce(grupo, true)`: só administradores do grupo podem enviar mensagens;
2. avisa no grupo que ele foi fechado até o bot responder todas as mensagens recebidas;
3. continua processando as rajadas já abertas desse grupo — elas fecham pela espera normal, porque ninguém mais além dos administradores consegue escrever;
4. quando **todas** as perguntas capturadas no grupo tiveram a sua primeira saída do [Fluxo de resposta](10_fluxo_de_resposta.md) — resposta, alternativas, cumprimento, silêncio de `_conversa` ou caso aberto —, chama `SetGroupAnnounce(grupo, false)` e avisa a reabertura.

Escolhas de alternativas e negativas só podem ser enviadas depois da reabertura; para as perguntas desse grupo, os prazos delas (`alternativas.expira_em`, `negativa.janela`) recomeçam a contar na reabertura, para que o autor tenha o prazo inteiro.

Para fechar o grupo o bot precisa ser administrador dele; se não for, registra o erro e avisa o grupo de administração.

### 2.3. Administração

O grupo para onde vão as dúvidas de confiança baixa e as que nenhuma tentativa resolveu. Os participantes desse grupo são os administradores do bot: qualquer mensagem ali é tratada como vinda de um administrador. O formato das mensagens e os comandos estão em [Fluxo §6](10_fluxo_de_resposta.md#6-grupo-de-administração).

## 3. Envio

- Toda resposta é enviada citando a mensagem de origem (`ContextInfo.StanzaID` + `Participant`), para que fique claro a quem se responde mesmo em grupos movimentados.
- Antes de enviar, o bot publica presença *digitando* no grupo por um intervalo proporcional ao tamanho do texto, limitado a poucos segundos, e espaça os envios para o mesmo grupo. Isso reduz o padrão de automação que leva o WhatsApp a banir números.

Próximo: [Fluxo de resposta →](10_fluxo_de_resposta.md)

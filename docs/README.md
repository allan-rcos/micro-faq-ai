# Documentação do Micro Faq AI

Índice: [← README](../README.md) | Próximo: [Visão geral →](01_visao_geral.md)

Leia na ordem abaixo: cada documento só usa conceitos definidos nos anteriores ou linka a definição no primeiro uso.

| # | Documento | O que cobre |
|---|-----------|-------------|
| 01 | [Visão geral](01_visao_geral.md) | O que o sistema faz, os dois componentes, o ciclo de aprendizado e o glossário |
| 02 | [Infraestrutura na OCI](02_infraestrutura.md) | As duas VMs, NoSQL, Object Storage, identidade e rede |
| 03 | [Build e deploy](03_build_e_deploy.md) | Compilação e minimização de tudo que vai para o servidor |
| 04 | [Normalização de texto](04_normalizacao.md) | O pipeline idêntico em Go e Python: Unicode, espaços, gírias (Aho-Corasick), frases e tokens |
| 05 | [Dados](05_dados.md) | Tabela única chave-valor (tópicos, tópicos reservados e casos), JSON de treinamento e layout do bucket |
| 06 | [Treinamento](06_treinamento.md) | Cron, dicionário estático, poda para o mini-dicionário, modelo PyTorch e exportação ONNX |
| 07 | [Pacote do modelo](07_pacote_modelo.md) | O `.bin` em Protocol Buffers: conteúdo, versionamento e nome no bucket |
| 08 | [Carga do modelo no bot](08_carga_do_modelo.md) | Como o Go descobre, testa e troca o pacote em execução, e como faz a inferência |
| 09 | [Bot e modos de grupo](09_bot_grupos.md) | Conexão `whatsmeow` e os modos *marcação*, *autônomo* e *administração* |
| 10 | [Fluxo de resposta](10_fluxo_de_resposta.md) | Tópicos reservados, faixas de confiança, alternativas, resposta negativa e fallback para os administradores |
| 11 | [Configuração](11_configuracao.md) | Referência dos arquivos TOML das duas VMs e do cron |
| 12 | [Ambiente local](12_ambiente_local.md) | O sistema inteiro na máquina do desenvolvedor, com Docker e implementações locais |
| 13 | [Questões em aberto](13_questoes_em_aberto.md) | Riscos que só a implementação e o uso vão confirmar |

## Changelog

| Documento | O que cobre |
|-----------|-------------|
| [Changelog](changelog/README.md) | Decisões trocadas, uma linha por decisão, com link para o desenho atual |

Os diagramas são gerados com PlantUML; os fontes e o comando de geração estão em [diagramas/](diagramas/README.md).

Próximo: [Visão geral →](01_visao_geral.md)

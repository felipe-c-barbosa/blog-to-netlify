# Content Engine Workflow

Este repositório continua sendo a última milha editorial do blog. A geração e a reescrita acontecem antes, no Marketing Brain e no `revisor-anti-ia`.

## Regra de entrada

Um draft só pode virar branch `cms/posts/{slug}` quando cumprir todos os pontos abaixo:

1. Brief preenchido no Marketing Brain.
2. Comparação com o corpus de estilo do Felipe.
3. `anti-ai-lint --profile felipe_barbosa --report --gate blog-publish` com gate aprovado.
4. Frontmatter pronto com `layout`, `title`, `author`, `permalink` e `categories`.

## Fluxo recomendado

1. Idear e priorizar no backlog editorial.
2. Montar o brief/evidence pack.
3. Gerar o draft em `temp/` no Marketing Brain.
4. Rodar o scorecard e o gate no `revisor-anti-ia`.
5. Criar a branch `cms/posts/{slug}`.
6. Salvar o post em `_posts/YYYY-MM-DD-{slug}.md`.
7. Abrir e revisar no DecapCMS.

## Revisão humana de 10-20 minutos

Use o tempo manual para:

- cortar gordura
- trocar abstração por caso real
- revisar a tese dos primeiros parágrafos
- revisar o fechamento
- decidir se publica ou se volta para reescrita

## O que não fazer aqui

- Não usar o Decap para "dar cara humana" a texto que ainda está genérico.
- Não criar branch de publicação antes do gate.
- Não tratar o `anti-ai-lint` como árbitro final; ele é um controle de qualidade, não um substituto da edição.

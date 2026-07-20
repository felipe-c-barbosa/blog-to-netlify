# Diagnóstico Técnico, SEO e AEO — felipebarbosa.me

**Data:** 2026-07-20
**Escopo:** revisão de código, implementação técnica, SEO/AEO e análises estratégicas. Nenhuma mudança foi implementada — este documento é o plano de melhorias.
**Método:** leitura completa do repositório + build local do site (`jekyll build`, Jekyll 4.3.4) para confirmar cada achado no output real de `_site/`.

---

## Sumário executivo

O site está saudável no geral: build rápido (~1,4s), boa base de Schema.org, `llms.txt` presente, sitemap e robots corretos, interlinking recente entre posts. Mas o build local revelou **bugs confirmados em produção** que anulam trabalho já feito:

1. **Os security headers nunca foram publicados** — o arquivo `_headers` é excluído do build pelo Jekyll (e, se fosse publicado, a CSP dele quebraria todo o analytics do site).
2. **A página `/startups/` está morta duas vezes** — sobrescrita por um redirect e com loop de categoria errado.
3. **O CMS grava categoria `product`, mas o site usa `produto`** — posts novos criados pelo Decap CMS não aparecem na página de categoria.
4. **`CONTENT_ENGINE.md` (workflow interno de geração de conteúdo com IA) está publicado no site público.**
5. **`llms.txt` aponta para uma URL que retorna 404.**

O plano abaixo está organizado em: bugs confirmados → SEO → AEO → performance → segurança → acessibilidade → análises estratégicas, com priorização ao final.

---

## 1. Bugs confirmados (verificados no build)

### 1.1 `_headers` não é publicado — site sem security headers ⚠️ crítico

Jekyll exclui do build todo arquivo iniciado com underscore. `_config.yml` não tem `include: ["_headers"]`, então o arquivo **não existe em `_site/`** (confirmado no build) e a Netlify nunca aplica esses headers. O site está em produção sem CSP, sem HSTS, sem X-Frame-Options.

Há um segundo problema encadeado: a CSP escrita no arquivo (`script-src 'self' 'unsafe-inline' fonts.googleapis.com cdn.jsdelivr.net`) **não inclui** `googletagmanager.com`, `js-na1.hs-scripts.com`, `cdn.segment.com`, `identity.netlify.com` nem `js.hsforms.net`. Se o arquivo simplesmente passasse a ser publicado, GA4, HubSpot (tracker + forms + chat), Segment e Netlify Identity seriam todos bloqueados.

**Correção:** adicionar `include: ["_headers"]` no `_config.yml` **e** reescrever a CSP com todos os domínios de terceiros realmente usados (ou movê-la para `netlify.toml` em `[[headers]]`, que não depende do Jekyll). Aproveitar para remover headers obsoletos: `X-XSS-Protection` e `X-UA-Compatible` são deprecados; `Accept-CH: DPR, Viewport-Width, Width` usa hints descontinuados.

### 1.2 `/startups/` morta e 3 posts órfãos

Dois defeitos combinados:

- `business.md` declara `redirect_from: /startups/`, e a página de redirect gerada **sobrescreve** a página real `startups.md` (confirmado: `_site/startups/index.html` contém `<meta http-equiv="refresh" ... url=/business/>`). Quem acessa `/startups/` é jogado para `/business/`.
- Mesmo sem o conflito, `startups.md` itera `site.categories.startup` (singular), mas os 3 posts usam `categories: startups` — a lista sairia vazia.

Resultado: 3 posts de startups não aparecem em nenhuma página de categoria. Além disso, `/startups/` aparece no sitemap como página real sendo um redirect.

**Correção:** decidir o destino — ou startups vira categoria de verdade (remover o redirect de `business.md` e corrigir o loop para `site.categories.startups`), ou os 3 posts são recategorizados como `business` e `startups.md` é removida.

### 1.3 Divergência de taxonomia entre CMS e site

`admin/config.yml` oferece as categorias `marketing`, `business`, `product`, `general`. Mas os posts existentes usam `produto` (14 posts) e a página `/produto/` itera `site.categories.produto`. Qualquer post novo criado pelo Decap CMS com categoria "Produto" recebe `product` e **não aparece em `/produto/`**.

A categoria `general` (1 post) não tem página nenhuma — e o link de categoria no meta do post aponta para `felipebarbosa.me/general` → 404.

**Correção:** padronizar a taxonomia em um conjunto único (sugestão: `marketing`, `produto`, `business`, `startups`) e alinhar CMS, posts e páginas de categoria. Recategorizar o post `general`.

### 1.4 Workflow interno de IA publicado no site público ⚠️ sensível

`CONTENT_ENGINE.md`, `README.md` e `LICENSE` são copiados para `_site/` e ficam acessíveis em `felipebarbosa.me/CONTENT_ENGINE.md` etc. (confirmado no build). O `CONTENT_ENGINE.md` descreve o pipeline de geração de conteúdo com IA, o `anti-ai-lint` e o "Marketing Brain" — informação estratégica e potencialmente delicada para a marca pessoal de quem assina os textos.

**Correção:** `exclude: ["CONTENT_ENGINE.md", "README.md", "LICENSE"]` no `_config.yml`.

### 1.5 `llms.txt` com link quebrado

O arquivo aponta `https://felipebarbosa.me/leituras/`, mas o permalink real é `/leituras-recentes/`. O link retorna 404 — justamente no arquivo feito para orientar crawlers de IA.

### 1.6 Request inútil em todos os posts

Em `_layouts/post.html:45`, o `<header>` sempre renderiza `background-image: url('...')`. Como praticamente nenhum post tem `feature-img`, o output é `background-image: url('/')` (confirmado no build) — o browser baixa o **HTML da homepage** como se fosse imagem em cada post visitado.

**Correção:** condicionar o atributo `style` à existência de `page.feature-img`.

### 1.7 Ícones fantasma na paginação

`index.html` usa `<i class="fa fa-chevron-left/right">`, mas o Font Awesome foi removido do projeto (substituído por SVGs inline). As tags renderizam vazias.

### 1.8 Meta verification duplicada

`head.html` tem **duas** tags `google-site-verification` diferentes (linhas 51–52). Uma delas provavelmente é de uma propriedade antiga do Search Console — verificar e remover a obsoleta.

### 1.9 Três portais HubSpot diferentes

- Tracker global no `<head>`: portal `50933491`
- Form de assinatura nos posts: portal `24045773`
- Form da página de contato: portal `49679780`

Isso significa que os leads/analytics estão fragmentados em três contas (ou dois desses IDs estão errados/mortos). Vale auditar qual é o portal ativo e unificar.

### 1.10 Feed declarado com MIME errado

O `head.html` anuncia `type="application/atom+xml"`, mas `feed.xml` é RSS 2.0. Funciona na prática, mas é inconsistente. Considerar migrar para o plugin `jekyll-feed` (Atom válido, `content:encoded`, mantido pela equipe do Jekyll) e manter `/feed.xml` como alias.

---

## 2. SEO

### 2.1 Meta descriptions: 68 de 68 posts sem `description`

Nenhum post tem `description` no front matter. O fallback usa o primeiro parágrafo truncado em 160 caracteres com reticências — funcional, mas raramente é o melhor snippet, e o CMS nem oferece o campo.

**Plano:** adicionar campo `description` (obrigatório) no `admin/config.yml`; escrever descriptions manuais começando pelos ~15 posts com mais tráfego/potencial (os guias "O que é..." recentes primeiro).

### 2.2 Dois `<h1>` por página

O header do site usa `<h1 class="site-title">` em toda página, somando-se ao `<h1>` do título do post (confirmado no build: 2×`<h1>` por post). Na home ainda há um terceiro no `header_text` do `_config.yml`.

**Plano:** manter `<h1>` apenas no conteúdo principal; trocar o site-title para `<p>`/`<div>` estilizado (ou `<h1>` apenas na home, se remover o do header_text).

### 2.3 OG image única para todos os posts

Sem `feature-img` nos posts, todos compartilham a mesma imagem OG genérica. Para um blog cuja distribuição é fortemente LinkedIn, isso reduz CTR de cada compartilhamento.

**Plano:** gerar OG images por post (título sobre template da marca). Dá para automatizar num passo de build (ex.: script que gera PNGs a partir do título) ou usar as transformações de texto do próprio Cloudinary via URL — sem serviço novo.

### 2.4 `dateModified` nunca muda

Nenhum post usa `last_modified_at`, então `article:modified_time` e `dateModified` do schema são sempre iguais à publicação. Para conteúdo evergreen de 2017–2021, sinalizar atualização real (após revisar o conteúdo) é um dos sinais mais baratos de relevância — para Google e para LLMs que citam fontes.

**Plano:** adotar `last_modified_at` no front matter (campo no CMS) e preenchê-lo a cada refresh de conteúdo. Nunca atualizar a data sem atualizar o conteúdo.

### 2.5 Links internos passando por redirect

- O footer compartilhado e o meta dos posts linkam para `https://www.felipebarbosa.me/...` — todo clique paga um 301 (o `netlify.toml` redireciona www→apex).
- O link de categoria no meta do post abre com `target="_blank"` (link interno abrindo nova aba, sem `rel="noopener"`).

**Plano:** trocar todos os links internos para URLs relativas ou apex; remover `target="_blank"` de links internos.

### 2.6 Canonical dos posts do Medium

As páginas em `/medium/` apontam canonical para `felipecbarbosa.medium.com/...`. Como você **tirou os posts do Medium**, se essas URLs de origem estiverem fora do ar o canonical aponta para 404 e o Google pode simplesmente não indexar as cópias locais.

**Plano:** verificar se as URLs do Medium ainda respondem. Se não (ou se a intenção é ranquear no seu domínio), remover o `canonical` desses arquivos e deixar o autorreferente.

### 2.7 Poluição no sitemap

Confirmado no sitemap gerado: 8 PDFs de certificados, arquivos internos e a página-redirect `/startups/`. Também há inconsistência de trailing slash (`/livros-para-profissionais/livros-startups` sem barra final, permalinks demais com barra).

**Plano:** excluir `files/` do sitemap (front matter defaults com `sitemap: false` ou exclusão do plugin), corrigir o conflito do 1.2 e padronizar permalinks com trailing slash.

### 2.8 Páginas de categoria fracas como hubs

As 6 páginas de categoria são quase idênticas ("Aqui você encontrará... :)" + lista de links). São a maior oportunidade de SEO estrutural do site: transformá-las em *pillar pages* — introdução real ao tema (500–1000 palavras), posts agrupados por subtema, links para os guias definitivos. Com 33 posts de marketing e 14 de produto, há material para hubs fortes de "Product Marketing" e "Gestão de Produto" em PT-BR.

Aproveitar para eliminar a duplicação de código: um único layout de categoria parametrizado em vez de 6 arquivos copiados.

### 2.9 Schema.org — bom, com ajustes

O que já existe é acima da média (BlogPosting completo, Person com `knowsAbout`, WebSite). Ajustes:

- `publisher` do BlogPosting é `Organization` com nome de pessoa — para blog pessoal, usar `Person` (ou o `@id` do Person já declarado no footer).
- Adicionar `BreadcrumbList` nos posts (Home → Categoria → Post).
- `/sobre/` merece `ProfilePage` referenciando o Person.
- Posts em formato pergunta ("O que é...", "Dúvidas comuns...") são candidatos naturais a seção FAQ + `FAQPage`/`Question` schema — relevante para featured snippets e para AEO (seção 3).

---

## 3. AEO / GEO (otimização para motores de resposta e LLMs)

O site já saiu na frente: `llms.txt` existe, robots libera crawlers de IA, os posts recentes seguem o padrão "definição direta no primeiro parágrafo". Próximos passos:

### 3.1 Corrigir e turbinar o `llms.txt`

- Corrigir o link quebrado (item 1.5).
- O arquivo é estático e lista só 5 páginas. Transformá-lo em template Liquid (como o `robots.txt` já é) que **itera todos os posts** com título, URL e descrição — vira um índice completo e sempre atualizado para LLMs. Opcionalmente gerar também `llms-full.txt` com conteúdo integral.

### 3.2 Atualizar a lista de crawlers no `robots.txt`

`anthropic-ai` é um user-agent desatualizado — o crawler atual da Anthropic é `ClaudeBot`. Adicionar os agentes relevantes de 2026: `ClaudeBot`, `Claude-Web`, `OAI-SearchBot`, `ChatGPT-User`, `PerplexityBot`, `Applebot-Extended`, `meta-externalagent`. (Manter os existentes; `Allow: /` para todos é coerente com a estratégia de ser citado.)

### 3.3 Conteúdo "citável"

LLMs citam fontes que respondem em blocos autocontidos. Práticas a padronizar no pipeline editorial (encaixa direto no CONTENT_ENGINE):

- Resposta de 40–60 palavras logo abaixo de cada H2 em formato de pergunta.
- Seção final de FAQ nos guias (com schema, item 2.9).
- Dados com fonte e data ("na pesquisa State of Product Marketing 2022 da PMA, 65%...") — já é o seu estilo; formalizar como regra.
- `dateModified` real (item 2.4) — LLMs e SGE dão peso forte a atualidade.

### 3.4 Autoridade de entidade

O Person schema já conecta LinkedIn/Twitter/GitHub. Duas adições baratas: incluir a URL do site no `sameAs` inverso (perfis apontando de volta) e manter consistência do cargo entre `llms.txt`, footer e `/sobre/` (hoje "Marketing Manager na Nubank" vs "Product Marketing Manager" no schema — divergência pequena, mas entidade consistente indexa melhor).

---

## 4. Performance

### 4.1 Quatro scripts de terceiros em toda página

O `<head>` carrega em **todas** as páginas: HubSpot tracker (+ chat), GA4, **Netlify Identity widget** e **Segment** (que ainda faz `analytics.page()`). Nos posts, soma-se o hsforms. Isso é provavelmente o maior custo de LCP/TBT do site — maior que qualquer otimização de CSS.

**Plano:**
- **Netlify Identity**: carregar apenas em `/admin/` (o snippet no `default.html` que redireciona após login também só faz sentido lá). Ganho imediato em todas as páginas.
- **Segment vs GA4 diretos**: hoje há tracking duplo. Ou o GA4 vira destination do Segment, ou o Segment sai (as páginas de desconto usam `analytics.track()` — migrar esses eventos se remover). Decidir por um.
- **HubSpot**: consolidar o portal (item 1.9) e avaliar carregar o chat sob interação (o padrão "facade") — o widget de chat é dos scripts mais pesados que existem.

### 4.2 Fontes

Google Fonts com 6 variações da Plus Jakarta Sans. `preconnect` e `display=swap` já estão certos. Otimizações: cortar pesos não usados (auditar se 500/600/800 aparecem no CSS) e considerar self-host (elimina a viagem a terceiros e o requisito de CSP extra).

### 4.3 Imagens

- Imagens dos posts (Cloudinary) sem `f_auto,q_auto,w_...` na maioria das URLs — o Cloudinary faz isso de graça via URL.
- Sem `loading="lazy"` e sem `width/height` (markdown puro) → CLS e download antecipado. Solução simples: um include/plugin de imagem, ou pós-processamento do HTML.
- Um posts referencia imagem em `miro.medium.com` — migrar para Cloudinary antes que o Medium quebre o hotlink.

### 4.4 CLS estrutural

O padding-top do `<body>` é calculado por JavaScript após o load (`header.offsetHeight`) — a página renderiza e "pula". Trocar por CSS (altura fixa do header ou `position: sticky`).

### 4.5 CSS/JS inline duplicado

O `shared-footer.html` injeta ~4KB de `<style>` em cada página. Mover para o SCSS do site (mantendo o footer como fonte canônica compartilhada, se ele é usado em outros sites, dá para manter o style inline apenas na versão exportada).

### 4.6 Cache

Sem headers de cache (consequência do 1.1). Ao republicar o `_headers`/`netlify.toml`, adicionar `Cache-Control` longo para `/css/*`, `/images/*` e fontes.

### 4.7 Dívidas de build

- Sass emite deprecations (`darken()`, `@import`) — quebrarão em versões futuras do dart-sass. Migrar para `color.adjust()`/`@use` quando conveniente.
- `jekyll-paginate` v1 é abandonado; `jekyll-paginate-v2` é o sucessor (e permitiria paginação por categoria nos hubs).
- Pin do Ruby: `netlify.toml` fixa 3.2.0; `.ruby-version` idem, mas está no `.gitignore` e commitado ao mesmo tempo — limpar essa ambiguidade.

---

## 5. Segurança

- **Headers ausentes em produção** — item 1.1, a correção mais importante desta seção.
- **Netlify Identity + git-gateway deprecados pela Netlify.** O backend do Decap CMS depende dos dois. Isso é um risco real de o `/admin/` parar de funcionar sem aviso. Migrar o Decap para backend `github` (OAuth) — e, feito isso, remover o Identity widget do site inteiro (item 4.1).
- `X-XSS-Protection` e `X-UA-Compatible` obsoletos no futuro `_headers` (remover ao reescrever).
- `/admin/` está com `Disallow` no robots (correto), mas confirme no Search Console que não foi indexado historicamente.

---

## 6. Acessibilidade

- **Dois `<h1>`** (item 2.2) — também é problema de a11y.
- **Sem skip-link** ("pular para o conteúdo") — com header fixo, leitores de teclado atravessam a navegação inteira em cada página.
- **`target="_blank"` sem `rel="noopener"`** nos links do meta do post (`post.html:50-51`).
- **Contraste do footer:** `#7a6e66` sobre `#150140` fica em ~3,8:1 — abaixo do mínimo AA (4,5:1) para o corpo de texto de 0,88rem. Clarear o cinza resolve.
- O menu mobile e os ícones sociais estão bem resolvidos (aria-expanded, aria-label, SVGs com aria-hidden) — manter o padrão.

---

## 7. Análises estratégicas (o que você talvez não esteja vendo)

### 7.1 Subdomínio da consultoria divide sua autoridade

`consultoria.felipebarbosa.me` é, para o Google, um site separado. Os 9 anos de autoridade do blog não fluem integralmente para lá. Se a consultoria é o objetivo comercial do ecossistema, a discussão subdirectory (`felipebarbosa.me/consultoria/`) vs subdomínio merece ser feita agora, antes que o subdomínio acumule backlinks próprios. Se o subdomínio ficar, mantenha o cross-linking forte (já existe) e considere schema `Service`/`Organization` lá apontando para o seu Person.

### 7.2 Content decay é seu maior risco de tráfego

Dois terços do acervo é de 2017–2021. Posts como "Product-Led Growth" (2019) e "Go-to-Market" (2021) tratam de temas onde o estado da arte mudou — e você já escreveu versões novas de alguns temas (ex.: o guia de GTM de 2026). Isso cria **canibalização interna**: dois posts seus competindo pela mesma keyword.

**Plano:** para cada tema duplicado, eleger o post canônico e fazer 301 do antigo (o `jekyll-redirect-from` já está instalado) ou atualizar o antigo e apontar o novo. Fazer um inventário keyword → post no Search Console para mapear as canibalizações reais.

### 7.3 A homepage não conta quem você é

A home é só a lista paginada de posts com um h1 genérico. Quem chega pela primeira vez (ou um LLM montando um perfil) não vê o posicionamento — "Marketing Manager no Nubank, instrutor na PM3, embaixador da PMA" está enterrado em `/sobre/`. Um bloco curto de apresentação acima da lista (com link para consultoria) transformaria a home no seu melhor cartão de visita, sem virar landing page.

### 7.4 Newsletter sem proposta de valor

O form do HubSpot aparece seco no fim dos posts, sem headline nem promessa ("receba X quando eu publicar Y"). Para quem tem a sua audiência de LinkedIn, a newsletter é o único canal próprio — vale copy dedicada, e talvez posicionar o form também no meio de posts longos.

### 7.5 Máquina editorial vs superfície do site

O CONTENT_ENGINE indica capacidade de produzir consistentemente. O gargalo de SEO hoje não é volume, é **estrutura**: taxonomia quebrada, hubs fracos, sem descriptions, sem refresh. Os itens deste diagnóstico rendem mais tráfego que os próximos 10 posts — e o pipeline pode incorporar as regras (description obrigatória, resposta direta pós-H2, FAQ, `last_modified_at`) como gate de publicação, junto do anti-ai-lint.

### 7.6 Código morto acumulado

`footer.html` (footer antigo, substituído pelo shared-footer), `disqus.html` (Disqus desativado), `search: false`, dezenas de ícones sociais não usados em `icons.html`, `_sass/includes/_footer.scss` do footer antigo, `medium.md` com lista manual duplicando os arquivos de `/medium/`. Nada disso quebra o site, mas confunde manutenção — uma limpeza única resolve.

---

## 8. Plano priorizado

### Fase 1 — Correções (1 sessão de trabalho, alto impacto, risco zero)
1. Publicar security headers com CSP correta (1.1) — preferencialmente via `netlify.toml`.
2. Excluir `CONTENT_ENGINE.md`, `README.md`, `LICENSE` do build (1.4).
3. Resolver `/startups/` e órfãos (1.2) + taxonomia CMS `product`→`produto` + post `general` (1.3).
4. Corrigir link do `llms.txt` (1.5) e atualizar crawlers do `robots.txt` (3.2).
5. `background-image` condicional no post (1.6), ícones `fa` (1.7), verification duplicada (1.8), MIME do feed (1.10).
6. Links internos sem www e sem `target="_blank"` (2.5) + `rel="noopener"` (seção 6).
7. Auditar os 3 portais HubSpot e unificar (1.9).

### Fase 2 — SEO on-page (1–2 semanas, em paralelo com publicação normal)
8. Campo `description` no CMS + descriptions dos principais posts (2.1).
9. H1 único (2.2) + skip-link + contraste do footer (seção 6).
10. Decidir canonical dos posts do Medium (2.6).
11. Limpar sitemap e padronizar permalinks (2.7).
12. Ajustes de schema: publisher Person, BreadcrumbList, ProfilePage (2.9).
13. `last_modified_at` no fluxo editorial (2.4).

### Fase 3 — Performance (1 sessão)
14. Identity só no `/admin/` + decisão Segment vs GA4 + chat sob demanda (4.1).
15. Cloudinary `f_auto,q_auto`, lazy loading, dimensões de imagem (4.3) + padding do header via CSS (4.4).
16. Cache headers (4.6) + footer styles no SCSS (4.5).

### Fase 4 — Estrutura e crescimento (contínuo)
17. Migrar Decap para backend GitHub antes que Identity morra (seção 5).
18. Hubs de categoria como pillar pages (2.8).
19. `llms.txt` dinâmico com índice completo de posts (3.1) + padrão de conteúdo citável e FAQ schema no pipeline (3.3).
20. OG images por post (2.3).
21. Inventário de canibalização + programa de refresh dos posts 2017–2021 (7.2).
22. Bloco de apresentação na home (7.3) + copy da newsletter (7.4).
23. Decisão subdomínio vs subdiretório da consultoria (7.1).
24. Limpeza de código morto (7.6) + migração Sass/paginate-v2 (4.7).

---

*Diagnóstico gerado a partir da revisão da branch `main` (commit `0a69b30`) com build local verificado.*

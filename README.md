# Site da Horizonte Benefícios

Duas pastas: `pronto/` é o site montado, pronto para subir; `fonte/` é de onde ele sai.

```
pronto/
  index.html               home completa
  area-do-associado.html   tela de acesso
fonte/
  build.py                 monta os dois arquivos acima
  head.part                <title>, fontes e toda a folha de estilo
  body.part                marcação da home
  admin-css.part           estilo do painel de conteúdo
  admin-html.part          marcação do painel
  admin-js.part            o painel em si (~830 linhas)
  login.html               tela de acesso, arquivo único
  imagens/                 os originais em alta
```

## Subir agora

Joga os dois arquivos de `pronto/` no servidor e acabou. Não há CSS solto, JS solto nem pasta de imagens: tudo está embutido em `data:` URI dentro do próprio HTML. A única coisa que vem de fora são as fontes Manrope e Sora, do Google Fonts — se o servidor não tiver saída para a internet, baixa os `.woff2`, hospeda junto e troca o `<link>` no topo do arquivo por um `@font-face`.

A home pesa 630 KB por causa das imagens embutidas. Se isso incomodar, separa as fotos em arquivos de verdade: no `build.py`, troca a função `uri()` por algo que copie a imagem para `pronto/imagens/` e devolva o caminho relativo. O resto continua igual.

## Mexer e reconstruir

```
cd fonte
python3 build.py
```

Só Python 3, sem instalar nada. Ele lê as partes, cola o painel dentro da home, embute as imagens e escreve os dois arquivos em `pronto/`.

Divisão das partes: `head.part` tem os tokens de cor (o bloco `:root`, com as variantes de tema escuro logo abaixo) e todo o CSS do site. `body.part` é a marcação, seção por seção, na ordem em que aparecem. Os três `admin-*.part` são o painel — o site funciona sem eles, é só tirar as três linhas correspondentes do `build.py`.

Os marcadores `__HERO__`, `__LOGO_COR__` e companhia são substituídos pelas imagens na hora do build. Se for trocar uma foto, troca o arquivo em `imagens/` mantendo o nome e roda o build de novo.

## O painel de conteúdo

O botão "Painel de conteúdo", no canto inferior direito, só aparece para quem tem permissão de edição. Quem visita o site não vê nada.

Uma coisa importante: **do jeito que está, o painel só grava dentro do Artifact do Claude.** Ele usa `window.claude.use('db')` para guardar o conteúdo e `window.claude.use('user')` para saber quem pode editar. Num servidor comum esses objetos não existem, então a primeira coisa que o script faz é desistir em silêncio — o site aparece normal, com o conteúdo que está escrito no HTML, e o botão do painel nunca surge. Nada quebra.

Para ligar o painel no seu servidor, o ponto de entrada é o final do `admin-js.part`:

```js
(async function () {
  if (!window.claude || !window.claude.use) return;
  var db = null, user = null;
  try { db = await window.claude.use('db'); } catch (e) {}
  try { user = await window.claude.use('user'); } catch (e) {}
  ...
  if (user && user.canEdit()) { D.pode = true; ... }
})();
```

São três contratos a cumprir, e nada além disso:

`db.doc(caminho).get()` devolve `{ c: <conteúdo> }` ou nulo. `db.doc(caminho).set({ c, em })` grava. `db.doc(caminho).delete()` apaga. Tem ainda `db.collection('hist').orderBy('em','desc').limit(10).get()`, usado só pela tela de histórico, e `db.doc(...).onSnapshot(fn)`, que pode virar uma função vazia se você não quiser atualização ao vivo.

`user.canEdit()` devolve true para quem pode editar. É o que decide se o botão aparece.

Os caminhos usados são `site/v4` (o que está publicado), `painel/rascunho4` (o rascunho, gravado sozinho ~1s depois da última tecla), `hist/<carimbo de tempo>` (uma cópia por publicação) e `imgc/<imagem>_<versão>_<n>` (as fotos, fatiadas em pedaços de 180 mil caracteres porque cada documento aceita 256 KB). Um endpoint PHP ou Node que leia e grave JSON por caminho resolve os quatro.

Se um dia quiser isso, me chama que eu adapto — é meia hora de trabalho.

### Como o painel enxerga a página

Não existe esquema duplicado: mexer no HTML muda o painel junto.

`data-admin="Nome"` numa seção a torna editável e a coloca no menu da esquerda. `data-admin-id="slug"` fixa a chave dela — é o que faz o conteúdo já publicado continuar batendo mesmo se você mudar a ordem das seções no HTML, então não troque esses valores depois de publicar. `data-admin-fixo` tira a seção do cartão de liga-desliga (é o caso do cabeçalho, do rodapé, da gaveta do celular, da barra fixa e do modal de cotação).

`data-lista="nome"` num contêiner libera adicionar, duplicar, mover e remover itens. Cada item guarda o próprio molde de HTML, porque eles não são iguais — o cartão da sede tem selo e endereço, os outros não.

`data-img`, `data-img-nome`, `data-img-pos` e `data-img-alpha` descrevem cada imagem: o nome é o que aparece no painel, `data-img-pos` libera as barras de enquadramento e `data-img-alpha` avisa que a imagem tem transparência (aí ela é comprimida em PNG, não em JPEG).

Os campos do painel são declarados no objeto `ESQ`, no meio do `admin-js.part`, por seletor CSS. Duas armadilhas que já me pegaram: `#pontos .sec-head p` casa com o `eyebrow`, que também é `<p>` e vem antes — o certo é `p:not(.eyebrow)`; e `.foot .bar span:last-child` casa com o span interno do próprio envelope de edição — o certo é `> span:last-child`. Se um campo aparecer com o texto errado ou não aparecer, é isso.

### Rascunho e publicação

O que você muda vira rascunho e não chega a ninguém. Ao fechar o painel com rascunho pendente, o site aparece com ele aplicado e uma tarja escura avisa embaixo — é a prévia, e só quem edita enxerga. "Revisar e publicar" lista o que vai ao ar antes de confirmar. O histórico guarda as dez últimas publicações com botão de restaurar.

O painel tem tokens de cor próprios, prefixados `--pn-`, escopados em `.pn`. Ele não herda a paleta da marca de propósito: roxo ali é só o realce da navegação e verde é só a ação de publicar, para a tela de edição não competir com o site.

## Área do associado

`area-do-associado.html` é maquete: o formulário valida os campos, mostra "Entrando…" e para por aí. Não autentica ninguém. Quando for ligar no sistema de verdade, o `<form>` está no fim do arquivo, com os campos `cpf` e `senha`.

## Detalhes que é bom não perder

Toda decoração de fundo — o anel da faixa de CTA, qualquer `::before` ou `::after` posicionado — leva `pointer-events: none`, e o bloco de conteúdo ao lado leva `position: relative` com `z-index: 1`. Sem essas duas linhas o pseudo-elemento cobre o botão em silêncio: o cursor vira mãozinha e o clique não chega. Foi exatamente o que aconteceu com o último botão da página.

Toda grade de uma coluna declara `minmax(0, 1fr)`. Sem isso, uma palavra longa dentro dela estica a página inteira e aparece rolagem horizontal.

A gaveta do menu mobile fica fora do `<header>`. O header tem `backdrop-filter`, e isso faz dele o bloco de referência de qualquer filho `position: fixed` — a gaveta dentro dele colapsava para uma linha.

O tema escuro acompanha o sistema. Os dois blocos de tokens escuros no `head.part` (`@media (prefers-color-scheme: dark)` e `:root[data-theme="dark"]`) precisam andar juntos: se mexer numa cor, mexe nos dois.

Testado de 320 a 1440 px de largura, sem rolagem horizontal em nenhuma, com todos os botões e links respondendo ao clique.

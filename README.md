# Lógica do quiz

Modelo do quiz usado nas `fa-fe6-hf-v3` e `fb-fe123-hf-v3`, pronto para levar para qualquer página ou produto. O [index.html](index.html) funciona sozinho: abra no navegador para ver o fluxo. Sem VSL configurada, as perguntas aparecem direto.

## O que ele faz

1. **Abre por cima de tudo.** Assim que a página carrega, o quiz cobre a página (modal em tela cheia com header, conteúdo e footer).
2. **Espera a VSL do quiz terminar.** Se houver uma VSL do vturb dentro de `.quiz-vsl`, a barra de progresso e as perguntas só aparecem quando o vídeo termina. Depois a tela rola até elas.
3. **Mostra uma pergunta por vez.** Ao clicar, a opção ganha borda e check verdes e as outras ficam cinza. Depois de 0,45s vem a próxima pergunta, e a barra avança.
4. **Grava a resposta na URL, se você quiser.** Uma pergunta com `data-param="variant"` grava a resposta na URL (ex.: `?variant=B`) sem recarregar a página.
5. **Roda o loading.** Depois da última pergunta, aparece um anel de 0 a 100% ("Calculating your personalized protocol..."), em uns 4s.
6. **Fecha e mostra a página.** A VSL do quiz sai do DOM (para de tocar), roda o `onFinish` e o quiz some.

## Antes de implementar: perguntas obrigatórias

Regra: antes de colocar o quiz numa página, pergunte tudo isto para quem pediu. Só comece a implementar com as respostas na mão. Não invente perguntas, cores, logo nem vídeo.

1. **Página:** em qual página (pasta/arquivo) o quiz entra? Qual é o container principal da página (vai no `pageSelector`)?
2. **Perguntas:** quais são as perguntas e as opções de cada uma? Se não houver, use as do modelo do [index.html](index.html) sem mudar o texto.
3. **Parâmetro na URL:** alguma resposta precisa ir para a URL (ex.: `?variant=`)? Qual pergunta e qual nome de parâmetro?
4. **VSL do quiz:** o quiz tem vídeo antes das perguntas? Se sim, qual o embed do vturb? Precisa ser um vídeo diferente da VSL da oferta.
5. **VSL da oferta:** a página tem VSL principal? Ela precisa carregar só depois do quiz (`loadOfferVsl()`)? Tem script de delay?
6. **Visual:** quais cores usar (ou seguir a identidade da página)? Tem layout de referência (Figma, print, outra página)?
7. **Logo:** qual arquivo de logo vai no header do quiz?
8. **Textos fixos:** título ("Personalized Protocol"), texto do loading e copyright do footer: manter ou trocar? Em qual idioma?
9. **Exit popup e checkout:** a página tem exit popup? O parâmetro precisa chegar no checkout (ver [Parâmetro na URL e checkout](#parâmetro-na-url-e-checkout-pagamerican))?

## Como implementar numa página

O `index.html` tem três blocos marcados com `===== Quiz: ... =====`.

1. **CSS:** copie o bloco do `<style>`, até o comentário "Só para este exemplo", para o CSS da página. Se o projeto usa SCSS compilado, cole no `.scss` e compile.
2. **HTML:** cole o `<div id="quiz">` como **primeiro elemento do `<body>`**. Ele já vem visível no HTML, então a página não "pisca" antes do quiz.
3. **JS:** cole o `<script>` do quiz antes do `</body>`, depois do HTML. Se preferir, coloque o conteúdo num `main.js` com `defer`.
4. **Página de trás:** ajuste `pageSelector` no `QUIZ_CONFIG` para o container principal da página (ex.: `#page-content`). Ele fica `inert` enquanto o quiz está aberto.
5. **Cores:** troque as variáveis `--quiz-*` no topo do CSS.
6. **Logo:** troque o logo dentro de `.quiz-header`.
7. **Opcional:** se a página tem exit popup, não deixe ele abrir com o quiz na tela. Basta checar `document.getElementById('quiz').hidden` antes de abrir.

### Perguntas

- Cada pergunta é uma `<section class="quiz-question">`. Só a primeira fica sem `hidden`.
- Cada opção é um `<button class="quiz-option" data-value="...">`.
- Pode ter quantas perguntas e opções quiser. A barra calcula o progresso sozinha.
- Mantenha o `<h3 tabindex="-1">` em cada pergunta: o foco vai para ele quando a pergunta troca (acessibilidade).
- Para gravar uma resposta na URL, coloque `data-param="nome"` na `<section>`. O valor gravado é o `data-value` da opção.

### Configuração (`QUIZ_CONFIG`)

| Campo | Para quê |
| --- | --- |
| `pageSelector` | Container da página que fica atrás do quiz. |
| `answerDelay` | Tempo (ms) mostrando a opção marcada antes de avançar. |
| `loadingSteps` | Trechos do loading: `[duração ms, % no fim]`. |
| `onFinish` | Roda quando o quiz fecha. Por padrão chama `loadOfferVsl()`, se existir. |

## VSLs do vturb (importante)

### VSL do quiz

Cole o embed completo do vturb (elemento + `<script>`) dentro de `<div class="quiz-vsl">`. Não precisa mexer em mais nada: o JS acha o `vturb-smartplayer` ali dentro e espera o evento `video:ended`.

- O vturb só emite `video:ended` (e `video:timeupdate`) na **reprodução de verdade**, depois que o cliente clica para ativar o som. A prévia muda que toca sozinha não libera o quiz.
- Não existe memória: se o cliente recarregar a página, precisa assistir até o fim de novo.

### VSL da oferta: carregar só depois do quiz

Se a página também tem uma VSL principal, carregue o `player.js` dela só quando o quiz fechar. Para isso, envolva o `<script>` do embed numa função:

```html
<vturb-smartplayer id="vid-XXXX" ...>...</vturb-smartplayer>
<script type="text/javascript">
  // O quiz chama no fim (onFinish): a VSL da oferta não toca escondida atrás do quiz.
  function loadOfferVsl() {
    var s = document.createElement("script");
    s.src = "https://scripts.converteai.net/.../players/XXXX/v4/player.js";
    s.async = true;
    document.head.appendChild(s);
  }
</script>
```

Se o script ficar "cru" (fora da função), a VSL da oferta começa a tocar atrás do quiz. Se o navegador liberar autoplay com som, ela toca por cima da VSL do quiz e o delay dela começa a contar antes da hora. Ao trocar o vídeo, troque só o ID e a URL e mantenha a função.

### Delay da oferta

O script de delay da oferta precisa apontar **só** para o player da oferta, porque o quiz também tem um `vturb-smartplayer` antes no DOM:

```js
var player = document.querySelector("#page-content vturb-smartplayer");
player.addEventListener("player:ready", function () {
  player.displayHiddenElements(540, ["[data-delay]"], { persist: true });
});
```

### Cuidados com o vturb

- **Nunca use o mesmo vídeo (mesmo ID) no quiz e na oferta.** O `player.js` do vturb roda uma vez e monta só o primeiro elemento com aquele ID. O segundo fica vazio e o delay da oferta nunca dispara.
- **O `persist: true` do `displayHiddenElements` é gravado por vídeo,** não por seletor (`displayHiddenElements-<id>`). Com vídeos diferentes no quiz e na oferta, não há conflito.

## Parâmetro na URL e checkout (PagAmerican)

O `s.js` da PagAmerican lê os parâmetros da URL **no carregamento da página** e envia essa foto em todos os cliques de sim/não. Um parâmetro gravado pelo quiz (ex.: `variant`) aparece na URL, mas **não entra no payload da PG**. Se a próxima página do funil precisar desse valor via PG, é preciso carregar o `s.js` só depois do quiz. Avalie o impacto antes: quem abandonar no quiz não registra a visita na PG.

## Checklist

- [ ] O CSS do quiz está carregado e o `[data-quiz-locked] { display: none }` está no `<head>`.
- [ ] O `<div id="quiz">` é o primeiro elemento do `<body>`.
- [ ] O `pageSelector` aponta para o container certo.
- [ ] A VSL do quiz está dentro de `.quiz-vsl`, com um ID diferente da VSL da oferta.
- [ ] O `player.js` da oferta está dentro de `loadOfferVsl()`.
- [ ] O script de delay da oferta usa um seletor escopado (`#page-content vturb-smartplayer`).
- [ ] Testado: as perguntas só aparecem no fim da VSL, o parâmetro entra na URL, o loading roda e a oferta aparece.

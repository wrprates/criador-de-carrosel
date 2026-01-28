<script>
  import html2canvas from 'html2canvas'
  import hljs from 'highlight.js'
  import { marked } from 'marked'
  import { onMount } from 'svelte'

  const renderer = new marked.Renderer()

  const escapeHtml = (value = '') =>
    String(value)
      .replace(/&/g, '&amp;')
      .replace(/</g, '&lt;')
      .replace(/>/g, '&gt;')
      .replace(/"/g, '&quot;')
      .replace(/'/g, '&#39;')

  const normalizeCode = (value) => {
    if (typeof value === 'string') return value
    if (value == null) return ''
    if (typeof value === 'object') {
      return String(value.text || value.raw || '')
    }
    return String(value)
  }

  const highlightCode = (code = '', language = '') => {
    const safeCode = normalizeCode(code)
    const validLanguage = language && hljs.getLanguage(language)
    if (validLanguage) {
      return hljs.highlight(safeCode, { language }).value
    }
    const { value } = hljs.highlightAuto(safeCode)
    return value || escapeHtml(safeCode)
  }

  renderer.code = (code = '', infostring = '', escaped) => {
    const rawCode = normalizeCode(code)
    const fallbackLanguage = typeof code === 'object' ? code?.lang || '' : ''
    const language = infostring.match(/\S+/)?.[0] ?? fallbackLanguage
    const autoHighlight = hljs.highlightAuto(rawCode)
    const validLanguage = language && hljs.getLanguage(language)
    const langForHighlight = (validLanguage ? language : autoHighlight.language || 'plaintext').replace(
      /[^a-z0-9\+\-#\.]/gi,
      ''
    )
    const highlighted = highlightCode(rawCode, language)
    const safeCode = highlighted || (escaped ? rawCode : escapeHtml(rawCode))
    const displayLang = (langForHighlight || 'code').toUpperCase()

    return `<pre class="code-block" data-lang="${displayLang}"><code class="hljs language-${langForHighlight}">${safeCode}</code></pre>`
  }

  marked.use({ renderer })
  marked.setOptions({ breaks: true, langPrefix: 'hljs language-' })

  const apiKey = import.meta.env.VITE_OPENAI_API_KEY

  const slideTemplates = [
    {
      id: 'slide-1',
      label: 'Slide 1 — Hook',
      description: 'Contextualize a dor e faça uma promessa direta.',
      type: 'hook',
      fallbackTitle: 'Comece com impacto',
      placeholder: `**Hook**\nPor que todo carrossel técnico precisa de sistemas?`
    },
    {
      id: 'slide-2',
      label: 'Slide 2 — Contexto',
      description: 'Explique rapidamente o cenário atual.',
      type: 'content',
      fallbackTitle: 'Contexto',
      placeholder: `**Diagnóstico 1**\nProfissionais muito técnicos ignoram ritmo visual.`
    },
    {
      id: 'slide-3',
      label: 'Slide 3 — Insight',
      description: 'Compartilhe dado ou aprendizado.',
      type: 'content',
      fallbackTitle: 'Insight',
      placeholder: `**Diagnóstico 2**\nSem tokens, cada criação começa do zero.`
    },
    {
      id: 'slide-4',
      label: 'Slide 4 — Framework',
      description: 'Liste passos ou pilares.',
      type: 'content',
      fallbackTitle: 'Pilar 1',
      placeholder: `**Pilar 1**\nTipografia fixa: 48px títulos, 28px subtítulos.`
    },
    {
      id: 'slide-5',
      label: 'Slide 5 — Framework',
      description: 'Continue com o próximo passo.',
      type: 'content',
      fallbackTitle: 'Pilar 2',
      placeholder: `**Pilar 2**\nGrid e espaçamentos fixos resolvem desalinhamentos.`
    },
    {
      id: 'slide-6',
      label: 'Slide 6 — Exemplo',
      description: 'Use um exemplo ou mini case.',
      type: 'content',
      fallbackTitle: 'Exemplo prático',
      placeholder: `**Exemplo prático**\nSlide com dados → badge + número + bullet direto.`
    },
    {
      id: 'slide-7',
      label: 'Slide 7 — Insight',
      description: 'Traga nuance, vantagem ou alerta.',
      type: 'content',
      fallbackTitle: 'Insight extra',
      placeholder: `**Insight extra**\nSem contraste, o leitor abandona no slide 3.`
    },
    {
      id: 'slide-8',
      label: 'Slide 8 — Lista',
      description: 'Liste bullets rápidos.',
      type: 'content',
      fallbackTitle: 'Checklist',
      placeholder: `**Checklist**\n- badge curto\n- CTA explícito\n- máximo 30 palavras`
    },
    {
      id: 'slide-9',
      label: 'Slide 9 — Quote',
      description: 'Feche a história com frase forte.',
      type: 'quote',
      fallbackTitle: 'Quote',
      placeholder: `**"Carrossel bom é o que parece simples, mas segue sistema rígido."**`
    },
    {
      id: 'slide-10',
      label: 'Slide 10 — CTA',
      description: 'Convide para o próximo passo.',
      type: 'cta',
      fallbackTitle: 'CTA',
      placeholder: `**Precisa de uma base pronta?**\nUse este criador pra manter consistência.`
    }
  ]

  const heroFontByType = {
    hook: 'clamp(2rem, 2.5vw, 3rem)',
    content: 'clamp(1.6rem, 2vw, 2.35rem)',
    quote: 'clamp(1.8rem, 2.2vw, 2.6rem)',
    cta: 'clamp(1.9rem, 2.4vw, 2.8rem)'
  }

  const totalSlides = slideTemplates.length

  let personaName = 'Wlademir Prates, PhD'
  let personaHandle = '@datamundo.cientista'
  let accentColor = '#111111'
  let backgroundColor = '#ffffff'
  let fontStack = 'Space Grotesk, Inter, system-ui, sans-serif'
  let ctaLabel = 'Segue meu perfil'
  let storyInput = ''
  let generating = false
  let generationError = ''
  let aiExtras = null
  let avatarSrc = '/avatar.jpg'
  let avatarEnabled = true
  let avatarMissing = false
  let hookVersions = []
  let selectedHookIndex = 0
  let promptMode = 'bio'

  let slidesContent = slideTemplates.map((template) => template.placeholder)
  let activeSlideIndex = 0
  let exporting = false
  let frameRefs = []
  let heroScale = 0.6
  let bodyScale = 1
  let headerScale = 1
  let codeScale = 1
  let slidesShowHero = slideTemplates.map(() => true)

  $: slides = slideTemplates.map((template, index) => ({
    ...template,
    content: slidesContent[index] ?? ''
  }))

  $: slidesWithParts = slides.map((slide, index) => ({
    ...slide,
    parts: splitContent(slide.content),
    showHero: slidesShowHero[index] ?? true
  }))

  $: activeSlide = slidesWithParts[activeSlideIndex] ?? slidesWithParts[0]

  function splitContent(block = '') {
    const lines = block.split('\n').map((line) => line.trim()).filter(Boolean)
    const hero = lines.shift() ?? ''
    return { hero, body: lines.join('\n') }
  }

  function renderBody(markdown) {
    return markdown?.trim() ? marked.parse(markdown) : ''
  }

  function renderInline(markdown) {
    return markdown?.trim() ? marked.parseInline(markdown) : ''
  }

  function updateSlideContent(index, value) {
    slidesContent = slidesContent.map((content, currentIndex) =>
      currentIndex === index ? value : content
    )
  }

  function updateShowHero(index, value) {
    slidesShowHero = slidesShowHero.map((currentValue, currentIndex) =>
      currentIndex === index ? value : currentValue
    )
  }

  function captureFrame(node, { index }) {
    frameRefs[index] = node
    return {
      destroy() {
        frameRefs[index] = null
      }
    }
  }

  onMount(async () => {
    try {
      const res = await fetch(avatarSrc, { method: 'HEAD' })
      if (!res.ok) {
        avatarMissing = true
        avatarSrc = ''
      }
    } catch (e) {
      avatarMissing = true
      avatarSrc = ''
    }
  })

  function handleAvatarUpload(event) {
    const file = event.target.files?.[0]
    if (!file) return
    const reader = new FileReader()
    reader.onload = (e) => {
      avatarSrc = e.target?.result || ''
      avatarMissing = false
      avatarEnabled = true
    }
    reader.readAsDataURL(file)
  }

  function focusSlide(index) {
    activeSlideIndex = index
  }

  const biographyPromptTemplate = `Você é um especialista em criar carrosséis virais para Instagram/LinkedIn seguindo padrões comprovados de alto engajamento.

ESTRUTURA OBRIGATÓRIA (10 SLIDES):
Slide 1: HOOK COM 3 VERSÕES (25-35 palavras cada) — NÃO use rótulos como "Hook 1". Apenas o texto do hook.
Slide 2: Origem + Contexto Inicial (30-35 palavras)
Slide 3: Primeiras Dificuldades + Obstáculos (30-35 palavras)
Slide 4: Descoberta/Momento Chave + Insight (30-35 palavras)
Slide 5: Luta/Crise + Ponto Baixo (30-35 palavras)
Slide 6: Persistência + Trabalho Duro (30-35 palavras)
Slide 7: Breakthrough + Primeira Vitória (30-35 palavras)
Slide 8: Resultado Final + Números Impressionantes (30-35 palavras)
Slide 9: Lição Universal + Filosofia (30-35 palavras)
Slide 10: CTA INSPIRADOR/MOTIVACIONAL com estrutura pedida.

Para os slides 2-10:
- Campo "hero": frase curta (máx. 5-8 palavras) que já seja uma mini-introdução chamativa ao corpo. Use verbo ou dado específico da história. NÃO use rótulos genéricos de entonação (ex.: nada de "Momento transformador", "Crise de confiança", "Persistência + Trabalho Duro", "Ponto crítico", "Virada", "Momento da virada", "Vitória inesperada", "Resultados impressionantes", "Lições aprendidas"). Não numere.
- Campo "body": texto (máx. 35 palavras) que desenvolve o hero, sem repetir o hero.

Após os 10 slides, inclua:
- Elementos Psicológicos Aplicados: 6 gatilhos mentais usados.
- Por que Este Carrossel Vai Inspirar/Viralizar: mínimo 5 razões específicas com dados da história; cite números/conquistas; explique apelo emocional.
- 3 Versões do Slide 1: explique brevemente a estratégia de cada hook (máx. 15 palavras cada).
- Diferencial Único/Poderoso/Devastador: 6+ diferenciais que tornam a história especial.

Regras: linguagem coloquial brasileira, frases curtas e impactantes, números específicos, CAPS pontual para ênfase, zero repetição, tom emocional progressivo, máximo 35 palavras nos slides 2-9, informações reais.
Nunca invente números ou fatos; evite mortes de outras pessoas; zero conteúdo político ou militante.

FORMATO DE RESPOSTA (JSON):
{
  "slides": [
    { "versions": ["v1","v2","v3"] },
    { "hero": "string", "body": "string ou lista de bullet" },
    ...
    { "hero": "string", "body": "cta final" }
  ],
  "elementos_psicologicos": ["..."],
  "por_que_vai_viralizar": ["..."],
  "versoes_slide_1": ["explicação das 3 versões do hook"],
  "diferenciais": ["..."]
}

Contexto da história: {INPUT}`

  const technicalPromptTemplate = `Você é um especialista em criação de carrosséis virais TÉCNICOS para Instagram e LinkedIn, dominando copy educacional, retenção visual e simplificação de conceitos complexos de dados/IA/estatística/programação.

Crie um carrossel técnico com 10 slides.

ESTRUTURA OBRIGATÓRIA (10 SLIDES):
Slide 1: HOOK COM 3 VERSÕES (25-35 palavras cada) — sem rótulos como "Hook 1".
  1) Contraste extremo
  2) Dado impressionante / verdade desconhecida
  3) Lição transformadora / promessa clara
Slide 2: Contexto / Por que importa (30-35 palavras) — problema, dor, relevância prática, onde aparece.
Slide 3: Definição simples e objetiva (máx. 35 palavras, sem jargão; metáfora curta ok).
Slide 4: Detalhamento técnico essencial (fórmula, função, trecho de código curto, estrutura ou componentes). Máx. 35 palavras.
Slide 5: Erros comuns / armadilhas.
Slide 6: Boas práticas / jeito certo.
Slide 7: Exemplo prático real (case, cenário, pipeline, dúvida clássica).
Slide 8: Resultado esperado / impacto (performance, tempo, clareza, negócio).
Slide 9: Lição final / síntese (até 35 palavras).
Slide 10: CTA inteligente (modelo fornecido).

Para os slides 2-10:
- Campo "hero": frase curta (máx. 5-8 palavras) que seja mini-introdução chamativa ao corpo, com verbo/dado específico. NÃO use rótulos genéricos de entonação (ex.: nada de "Momento transformador", "Crise de confiança", "Persistência + Trabalho Duro", "Ponto crítico", "Virada", "Momento da virada", "Vitória inesperada", "Resultados impressionantes", "Lições aprendidas"). Não numere.
- Campo "body": texto (máx. 35 palavras) que desenvolve o hero, sem repetir o hero.

Após os 10 slides, inclua:
- Elementos Psicológicos Usados: 5 a 7 (clareza, autoridade, contraste, expectativa, identificação, recompensa, simplicidade técnica etc.).
- Por que Este Carrossel Vai Viralizar: 5 motivos claros (utilidade prática, conceito difícil simplificado, aplicabilidade imediata, impacto na carreira, retenção visual).
- 3 Versões do Slide 1 — Estratégia: intenção de cada hook (contraste, dado, transformação) em máx. 15 palavras cada.
- Diferenciais Únicos: 6+ pontos (profundidade técnica, explicação acessível, código real, alta aplicabilidade, narrativa educacional fluida etc.).

Regras: 30–35 palavras por slide (2–9); clareza total; foco em valor prático; código só se ajudar; fórmulas só se essenciais; frases curtas; linguagem brasileira, humana, simples; tom conversacional. ZERO números ou fatos inventados. Zero política.

FORMATO DE RESPOSTA (JSON):
{
  "slides": [
    { "versions": ["v1","v2","v3"] },
    { "hero": "string", "body": "string ou lista de bullet" },
    ...
    { "hero": "string", "body": "cta final" }
  ],
  "elementos_psicologicos": ["..."],
  "por_que_vai_viralizar": ["..."],
  "versoes_slide_1": ["explicação das 3 versões do hook"],
  "diferenciais": ["..."]
}

Tema técnico: {INPUT}`

  const densePromptTemplate = `Você é um especialista em criar carrosséis de CONTEÚDO DENSO e DIDÁTICO para Instagram/LinkedIn. Foque em profundidade, utilidade prática e clareza. Use bullets e listas numeradas sempre que ajudarem a ler rápido. Um bloco de código curto (6-12 linhas ou pseudocódigo) é obrigatório em um slide técnico.

ESTRUTURA OBRIGATÓRIA (10 SLIDES):
Slide 1: HOOK com 3 versões (35-50 palavras cada) — contraste forte, dado surpreendente ou tese polêmica; sem rótulos como "Hook 1".
Slide 2: Panorama + Por que importa (45-60 palavras) — explique relevância e onde o problema aparece; inclua 2-3 bullets numerados se ajudar.
Slide 3: Tese central + definição clara (45-60 palavras) — conceito em linguagem simples, metáfora curta opcional.
Slide 4: Framework/Passos numerados (45-60 palavras) — 3 a 5 passos objetivos com verbos de ação.
Slide 5: Profundidade conceitual (45-60 palavras) — detalhe nuances, limites, trade-offs; cite quando NÃO usar.
Slide 6: Código/Algoritmo sintético (até 12 linhas) — forneça bloco de código curto ou pseudocódigo + explicação de 20-25 palavras. Use comentários curtos.
Slide 7: Exemplos aplicados (45-60 palavras) — 2-4 cenários reais ou cases rápidos.
Slide 8: Métricas e impacto (45-60 palavras) — números realistas, ganhos/risco; compare "antes vs depois" em 2 bullets.
Slide 9: Armadilhas + Boas práticas (45-60 palavras) — liste DOs/DON'Ts; priorize ações concretas.
Slide 10: CTA prático (25-35 palavras) — convide para aplicar; cite tempo/benefício; evite genérico.

Para os slides 2-10:
- Campo "hero": frase curta (5-9 palavras) específica do assunto, com verbo/dado concreto. NÃO use rótulos genéricos de entonação (ex.: nada de "Momento transformador", "Crise", "Ponto crítico", "Virada", "Resultados impressionantes", "Lições aprendidas").
- Campo "body": 45-60 palavras (25-35 no CTA) desenvolvendo o hero; use bullets ou listas numeradas quando fizer sentido; mantenha clareza e densidade. Em listas, prefixe com "-" ou "1)" etc.

Após os 10 slides, inclua:
- Elementos Psicológicos Aplicados: 6-8 (clareza, autoridade, contraste, expectativa, simplicidade técnica, especificidade, evidência numérica, benefício imediato).
- Por que Este Carrossel Vai Viralizar: mínimo 5 motivos específicos ancorados no assunto (valor prático, profundidade rara, código pronto, aplicabilidade imediata, comparações claras).
- 3 Versões do Slide 1 — Estratégia: intenção de cada hook (contraste, dado, tese polêmica) em máx. 15 palavras cada.
- Diferenciais Únicos: 6+ pontos (densidade + clareza, frameworks acionáveis, exemplos reais, código direto, bullets numerados, armadilhas explícitas).

Regras: linguagem brasileira direta; zero floreio narrativo; foco em informação acionável; frases curtas; nunca invente dados; se não houver número realista, use intervalo prudente (ex.: "economia de 5-10%").

FORMATO DE RESPOSTA (JSON):
{
  "slides": [
    { "versions": ["v1","v2","v3"] },
    { "hero": "string", "body": "string ou lista de bullet" },
    ...
    { "hero": "string", "body": "cta final" }
  ],
  "elementos_psicologicos": ["..."],
  "por_que_vai_viralizar": ["..."],
  "versoes_slide_1": ["explicação das 3 versões do hook"],
  "diferenciais": ["..."]
}

Tema/assunto: {INPUT}`

  const sevenItemsPromptTemplate = `Você é um especialista em criar carrosséis técnicos, profissionais e densos para Instagram/LinkedIn. O post sempre lista 7 itens (ex.: "7 funções do tidyverse", "7 pacotes Python", "7 passos para entrevista técnica"). Sempre responda em JSON válido no formato abaixo.

ESTRUTURA (10 SLIDES):
1) Hook inicial (3 versões, 30–45 palavras cada) — contraste forte ou promessa clara sobre os 7 itens; sem rótulos como "Hook 1".
2) Enquadramento: por que esses 7 itens importam, critério de escolha, quem ganha com isso (40–55 palavras).
3) Item 1 — o que é, quando usar, mini exemplo (35–50 palavras; bullets opcionais).
4) Item 2 — idem (35–50 palavras).
5) Item 3 — idem (35–50 palavras).
6) Item 4 — idem (35–50 palavras).
7) Item 5 — idem (35–50 palavras).
8) Item 6 — idem (35–50 palavras).
9) Item 7 — idem (35–50 palavras).
10) Fechamento + CTA final (25–35 palavras): síntese do ganho e chamada explícita para me seguir se quer ganhar 20k+/mês em dados.

Regras de escrita:
- Hero de cada slide: 4–8 palavras, verbo/dado específico; nada de rótulos genéricos ("Virada", "Crise", "Lições" etc.).
- Body: frases curtas; bullets/numeração quando ajudam; foco em clareza técnica e aplicabilidade.
- Se código ajudar, use um snippet curto (6–10 linhas) em UM dos itens, com 1–2 comentários claros.
- Nunca invente dados; se precisar, use intervalo plausível ("ganho de 5–10%").
- Português direto, profissional, sem floreio narrativo.

FORMATO DE RESPOSTA (JSON):
{
  "slides": [
    { "versions": ["v1","v2","v3"] },
    { "hero": "string", "body": "string ou lista de bullet" },
    { "hero": "string", "body": "string ou lista de bullet" },
    { "hero": "string", "body": "string ou lista de bullet" },
    { "hero": "string", "body": "string ou lista de bullet" },
    { "hero": "string", "body": "string ou lista de bullet" },
    { "hero": "string", "body": "string ou lista de bullet" },
    { "hero": "string", "body": "string ou lista de bullet" },
    { "hero": "string", "body": "string ou lista de bullet" },
    { "hero": "string", "body": "cta final para seguir e buscar 20k+/mês em dados" }
  ],
  "elementos_psicologicos": ["clareza", "autoridade", "contraste", "especificidade", "benefício imediato", "..."],
  "por_que_vai_viralizar": ["5 motivos específicos ancorados no tema"],
  "diferenciais": ["6+ pontos fortes do conteúdo"]
}

Tema/assunto: {INPUT}`

  const cheatSheetPromptTemplate = `Você é um especialista em criar carrosséis CHEAT SHEET densos e profissionais (Instagram/LinkedIn). Conteúdo = consulta rápida, sem superficialidade. Cada slide deve trazer densidade real (múltiplos comandos, flags, exemplos concretos). Sempre responda em JSON válido no formato abaixo.

ESTRUTURA (10 SLIDES):
1) Hook (3 versões, 25–40 palavras cada) — promessa de velocidade/precisão com este cheat; sem rótulos como "Hook 1".
2) Escopo e pré-reqs (40–60 palavras): o que cobre, público-alvo, ambiente/ferramentas, limitações.
3) Bloco 1 — comandos/operadores essenciais (45–70 palavras; 4–6 bullets com sintaxe + uso).
4) Bloco 2 — padrões frequentes/combinações típicas (45–70 palavras; 3–5 bullets com entrada/saída esperada).
5) Bloco 3 — casos rápidos com parâmetros (45–70 palavras; 3–5 bullets, cada um com comando + efeito).
6) Bloco 4 — erros/armadilhas (45–70 palavras; 4–6 bullets DO NOT / evitar, cite causa/efeito).
7) Bloco 5 — boas práticas (45–70 palavras; 4–6 bullets DO com ação concreta e resultado).
8) Bloco 6 — snippet curto (6–12 linhas) com 1–2 comentários; abaixo, 25–40 palavras explicando entradas/saídas e por que funciona.
9) Resumo/mini tabela mental (40–60 palavras; pode ser bullets "quando usar X / evitar Y / alternativa Z").
10) CTA final (25–35 palavras): siga para mais cheat sheets e para chegar a 20k+/mês em dados.

Regras de escrita:
- Hero de cada slide: 4–8 palavras, verbo/dado específico; nada de rótulos genéricos ("Virada", "Crise", "Lições" etc.).
- Body: densidade máxima; evite frases vazias. Use bullets numerados ou com prefixo claro; inclua sintaxe exata, flags, entradas/saídas, limites.
- Se não houver dado real, use intervalo plausível ("ganho de 5–10%"); nunca invente valores absolutos sem base.
- Português direto, profissional; zero floreio narrativo; priorize comandos, padrões, trade-offs.

FORMATO DE RESPOSTA (JSON):
{
  "slides": [
    { "versions": ["v1","v2","v3"] },
    { "hero": "string", "body": "string ou lista de bullet" },
    { "hero": "string", "body": "string ou lista de bullet" },
    { "hero": "string", "body": "string ou lista de bullet" },
    { "hero": "string", "body": "string ou lista de bullet" },
    { "hero": "string", "body": "string ou lista de bullet" },
    { "hero": "string", "body": "string ou lista de bullet" },
    { "hero": "string", "body": "string ou lista de bullet" },
    { "hero": "string", "body": "string ou lista de bullet" },
    { "hero": "string", "body": "cta final para seguir e buscar 20k+/mês em dados" }
  ],
  "elementos_psicologicos": ["clareza", "autoridade", "contraste", "especificidade", "benefício imediato", "..."],
  "por_que_vai_viralizar": ["5 motivos específicos ancorados no tema"],
  "diferenciais": ["6+ pontos fortes do conteúdo"]
}

Tema/assunto: {INPUT}`

  const promptModeConfig = {
    bio: {
      helper:
        'Conte o resumo da história. A chave vem de .env.local como VITE_OPENAI_API_KEY (não comitar).',
      placeholder: 'Ex: Jovem de 17 anos que vendia brigadeiro na escola virou referência em IA...',
      missingInput: 'Descreva a história em um parágrafo curto antes de gerar.',
      inputLabel: 'Contexto da história'
    },
    tech: {
      helper:
        'Descreva o tema técnico/conceito. A chave vem de .env.local como VITE_OPENAI_API_KEY (não comitar).',
      placeholder: 'Ex: Como usar shap values no XGBoost para explicar previsões de churn...',
      missingInput: 'Descreva o tema técnico em um parágrafo curto antes de gerar.',
      inputLabel: 'Tema técnico'
    },
    dense: {
      helper:
        'Explique o assunto a ser destrinchado em 10 slides densos; cite objetivo e público.',
      placeholder:
        'Ex: Prompt engineering para analistas de marketing: técnicas, exemplos de prompts, erros comuns, métrica de sucesso...',
      missingInput: 'Descreva o tema, objetivo e público para gerar o conteúdo denso.',
      inputLabel: 'Tema do conteúdo denso'
    },
    seven: {
      helper:
        'Defina o tema com 7 itens que serão explicados em 10 slides (1 hook, 1 enquadramento, 7 itens, 1 CTA).',
      placeholder:
        'Ex: 7 funções essenciais do tidyverse para BI rápido; público: analistas júnior; objetivo: acelerar dashboards.',
      missingInput: 'Descreva o tema e os 7 itens (ou o recorte) para gerar o carrossel.',
      inputLabel: 'Tema com 7 itens'
    },
    cheat: {
      helper:
        'Defina o tema do cheat sheet e público (ex.: comandos Git para cientistas de dados).',
      placeholder:
        'Ex: Cheat sheet de pandas para data viz rápida (select, groupby, melt, pivot, plot). Público: analistas junior.',
      missingInput: 'Descreva o tema e o recorte do cheat sheet para gerar o carrossel.',
      inputLabel: 'Tema do cheat sheet'
    }
  }

  let currentPromptConfig = promptModeConfig[promptMode] ?? promptModeConfig.bio

  $: currentPromptConfig = promptModeConfig[promptMode] ?? promptModeConfig.bio

  function buildPrompt(story) {
    const templateMap = {
      bio: biographyPromptTemplate,
      tech: technicalPromptTemplate,
      dense: densePromptTemplate,
      seven: sevenItemsPromptTemplate,
      cheat: cheatSheetPromptTemplate
    }
    const template = templateMap[promptMode] ?? biographyPromptTemplate
    return template.replaceAll('{INPUT}', story)
  }

  const bannedHeroTerms = [
    'momento da virada',
    'momento transformador',
    'ponto crítico',
    'ponto critico',
    'crise de confiança',
    'crise',
    'virada',
    'trabalho e superação',
    'trabalho e superacao',
    'descoberta',
    'momento chave',
    'superação',
    'superacao',
    'vitória inesperada',
    'vitoria inesperada',
    'resultados impressionantes',
    'lições aprendidas',
    'licoes aprendidas'
  ]

  function sanitizeHero(heroText = '', body = '') {
    const lower = heroText.toLowerCase()
    const isGeneric = bannedHeroTerms.some((term) => lower.includes(term))
    if (!isGeneric) return heroText

    const firstSentence = body.split(/[\.\!\?]/)[0]?.trim() ?? ''
    const fallback = firstSentence
      .split(' ')
      .filter(Boolean)
      .slice(0, 8)
      .join(' ')

    return fallback || heroText
  }

  function formatSlideContent(slideData = {}, template) {
    if (!slideData) return template.placeholder
    const { title, hero, body, versions } = slideData

    if (versions?.length) {
      const hooks = versions.map((text) => text).join('\n')
      return ['Hooks (3 opções)', hooks].filter(Boolean).join('\n')
    }

    const normalizedBody = Array.isArray(body) ? body.join('\n') : body ?? ''
    const heroText = sanitizeHero(hero || title || template.fallbackTitle, normalizedBody)

    return [heroText, normalizedBody].filter(Boolean).join('\n')
  }

  function applyAISlides(slidesData = []) {
    slidesContent = slideTemplates.map((template, index) =>
      formatSlideContent(slidesData[index], template)
    )
    if (hookVersions?.length) {
      slidesContent = slidesContent.map((content, currentIndex) =>
        currentIndex === 0 ? hookVersions[selectedHookIndex] ?? content : content
      )
    }
    slidesShowHero = slidesShowHero.map(() => true)
  }

  function mapAIExtras(parsed) {
    aiExtras = {
      elementos: parsed?.elementos_psicologicos ?? [],
      razoes: parsed?.por_que_vai_viralizar ?? [],
      versoesHook: parsed?.versoes_slide_1 ?? [],
      diferenciais: parsed?.diferenciais ?? []
    }
  }

  function selectHookOption(index) {
    selectedHookIndex = index
    slidesContent = slidesContent.map((content, currentIndex) =>
      currentIndex === 0 ? hookVersions[index] ?? content : content
    )
  }

  async function generateWithAI() {
    if (!storyInput.trim()) {
      generationError = currentPromptConfig?.missingInput ?? 'Preencha o campo antes de gerar.'
      return
    }

    if (!apiKey) {
      generationError = 'Defina VITE_OPENAI_API_KEY no seu .env.local (não comitar).'
      return
    }

    generationError = ''
    generating = true

    try {
      const response = await fetch('https://api.openai.com/v1/chat/completions', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          Authorization: `Bearer ${apiKey}`
        },
        body: JSON.stringify({
          model: 'gpt-4o-mini',
          temperature: 0.7,
          response_format: { type: 'json_object' },
          messages: [
            { role: 'system', content: 'Você é um escritor de carrosséis virais e SEMPRE responde em JSON válido conforme o formato pedido.' },
            { role: 'user', content: buildPrompt(storyInput.trim()) }
          ]
        })
      })

      if (!response.ok) {
        throw new Error('Falha ao chamar a API. Confirme a chave e o saldo.')
      }

      const payload = await response.json()
      const content = payload?.choices?.[0]?.message?.content
      const parsed = JSON.parse(content ?? '{}')

      if (parsed?.slides?.length) {
        hookVersions = parsed?.slides?.[0]?.versions ?? []
        selectedHookIndex = 0
        applyAISlides(parsed.slides)
      }

      mapAIExtras(parsed)
      focusSlide(0)
    } catch (error) {
      generationError = error?.message ?? 'Erro desconhecido ao gerar carrossel.'
    } finally {
      generating = false
    }
  }


  const TARGET_WIDTH = 1080
  const TARGET_HEIGHT = 1350
  const MIN_SCALE = 2

  async function exportSlide(index) {
    const node = frameRefs[index]
    if (!node) return
    const rect = node.getBoundingClientRect()
    const baseScale = Math.max(TARGET_WIDTH / rect.width, TARGET_HEIGHT / rect.height)
    const scale = Math.max(baseScale, MIN_SCALE)

    const canvas = await html2canvas(node, {
      backgroundColor,
      scale
    })

    const link = document.createElement('a')
    link.download = `carrossel-${String(index + 1).padStart(2, '0')}.png`
    link.href = canvas.toDataURL('image/png')
    link.click()
  }

  async function exportSelectedSlide() {
    if (activeSlideIndex < 0) return
    exporting = true
    await exportSlide(activeSlideIndex)
    exporting = false
  }

  async function exportAllSlides() {
    exporting = true
    for (let index = 0; index < slides.length; index += 1) {
      await exportSlide(index)
    }
    exporting = false
  }
</script>

<main class="app-grid">
  <section class="panel">
    <header class="panel__header">
      <div>
        <p class="eyebrow">Criador v1</p>
        <h1>Carrossel padronizado</h1>
        <p class="muted">Cole o texto de cada slide (Markdown simples) e exporte em 4:5 com tipografia consistente.</p>
      </div>
    </header>

    <div class="panel__section">
      <h2>Geração com OpenAI</h2>
      <p class="muted small">
        {currentPromptConfig.helper}
      </p>
      <label class="sr-only" for="story-input">
        {currentPromptConfig.inputLabel}
      </label>
      <textarea
        id="story-input"
        rows="4"
        placeholder={currentPromptConfig.placeholder}
        bind:value={storyInput}
      ></textarea>
      <div class="generation-actions">
        <button class="primary" on:click={generateWithAI} disabled={generating}>
          {generating ? 'Gerando…' : 'Gerar carrossel com IA'}
        </button>
        <span class="muted tiny">{apiKey ? 'Chave detectada (env)' : 'Sem chave carregada'}</span>
      </div>
      {#if generationError}
        <p class="tiny error">{generationError}</p>
      {/if}
    </div>

    <div class="panel__section">
      <h2>Identidade</h2>
      <div class="form-grid">
        <label>
          <span>Tipo de carrossel</span>
          <select bind:value={promptMode}>
            <option value="bio">Narrativa / Bio</option>
            <option value="tech">Técnico</option>
            <option value="dense">Conteúdo denso</option>
            <option value="seven">Lista de 7 itens (10 slides)</option>
            <option value="cheat">Cheat sheet</option>
          </select>
        </label>
        <label>
          <span>Nome</span>
          <input type="text" bind:value={personaName} />
        </label>
        <label>
          <span>@handle</span>
          <input type="text" bind:value={personaHandle} />
        </label>
        <label>
          <span>Cor principal</span>
          <input type="color" bind:value={accentColor} />
        </label>
        <label>
          <span>Fundo</span>
          <input type="color" bind:value={backgroundColor} />
        </label>
        <label class="full">
          <span>Pilha de fontes</span>
          <input type="text" bind:value={fontStack} />
        </label>
        <label class="full">
          <span>Texto do CTA</span>
          <input type="text" bind:value={ctaLabel} />
        </label>
        <label class="full">
          <span>Mostrar avatar</span>
          <div class="inline-control">
            <input
              id="avatar-toggle"
              type="checkbox"
              checked={avatarEnabled}
              on:change={(event) => (avatarEnabled = event.currentTarget.checked)}
            />
            <label for="avatar-toggle">Exibir avatar ao lado do handle</label>
          </div>
        </label>
        <label class="full">
          <span>Avatar (usa /public/avatar.jpg por padrão)</span>
          <input type="file" accept="image/*" on:change={handleAvatarUpload} />
          {#if avatarMissing}
            <p class="tiny error">Arquivo avatar.jpg não encontrado. Faça upload aqui.</p>
          {/if}
        </label>
        <label class="full range-control">
          <span>Escala título ({heroScale.toFixed(2)}x)</span>
          <input
            type="range"
            min="0.3"
            max="1.25"
            step="0.05"
            bind:value={heroScale}
          />
        </label>
        <label class="full range-control">
          <span>Escala corpo ({bodyScale.toFixed(2)}x)</span>
          <input
            type="range"
            min="0.35"
            max="1.25"
            step="0.05"
            bind:value={bodyScale}
          />
        </label>
        <label class="full range-control">
          <span>Escala header ({headerScale.toFixed(2)}x)</span>
          <input
            type="range"
            min="0.35"
            max="1.25"
            step="0.05"
            bind:value={headerScale}
          />
        </label>
        <label class="full range-control">
          <span>Escala código ({codeScale.toFixed(2)}x)</span>
          <input
            type="range"
            min="0.35"
            max="1.5"
            step="0.05"
            bind:value={codeScale}
          />
        </label>
      </div>
    </div>

    {#if hookVersions?.length}
      <div class="panel__section">
        <h2>Escolha o hook</h2>
        <p class="muted small">3 opções geradas. Escolha uma para usar como Slide 1.</p>
        <div class="hook-options">
          {#each hookVersions as hook, index}
            <label class={`hook-card ${selectedHookIndex === index ? 'hook-card--active' : ''}`}>
              <div class="hook-card__header">
                <input
                  type="radio"
                  name="hook-option"
                  id={`hook-option-${index}`}
                  value={index}
                  checked={selectedHookIndex === index}
                  on:change={() => selectHookOption(index)}
                />
                <span>Hook {index + 1}</span>
              </div>
              <p>{hook}</p>
            </label>
          {/each}
        </div>
      </div>
    {/if}

    <div class="panel__section">
      <h2>Conteúdo dos 10 slides</h2>
      <p class="muted small">Use Markdown básico: <code>**negrito**</code>, <code>*itálico*</code>, <code>- listas</code>. A primeira linha vira o título.</p>
      <div class="slides-grid">
        {#each slidesWithParts as slide, index}
          <div class="slide-input" class:slide-input--active={index === activeSlideIndex}>
            <div class="slide-input__header">
              <div>
                <p class="slide-index">Slide {index + 1}</p>
                <span>{slide.label}</span>
              </div>
              <button type="button" class="ghost small" on:click={() => focusSlide(index)}>
                Ver preview
              </button>
            </div>
            <label class="sr-only" for={`slide-input-${index}`}>Conteúdo do slide {index + 1}</label>
            <textarea
              id={`slide-input-${index}`}
              rows="4"
              value={slidesContent[index]}
              placeholder={slide.placeholder}
              on:focus={() => (activeSlideIndex = index)}
              on:input={(event) => updateSlideContent(index, event.target.value)}
            ></textarea>
            <div class="slide-meta">
              <span class="muted tiny">{slide.description}</span>
              <label class="slide-toggle">
                <input
                  type="checkbox"
                  checked={slidesShowHero[index]}
                  on:change={(event) => updateShowHero(index, event.currentTarget.checked)}
                />
                <span>Mostrar título</span>
              </label>
            </div>
          </div>
        {/each}
      </div>
    </div>
  </section>

  <section class="preview">
    <div class="preview__header">
      <div>
        <p class="eyebrow">Preview</p>
        <h2>Carrossel ({totalSlides} slides)</h2>
      </div>
      <div class="preview__actions">
        <button class="ghost" on:click={exportSelectedSlide} disabled={exporting}>
          Exportar selecionado
        </button>
        <button class="primary" on:click={exportAllSlides} disabled={exporting}>
          {exporting ? 'Exportando…' : 'Exportar tudo'}
        </button>
      </div>
      </div>

      <div class="frames">
        {#each slidesWithParts as slide, index}
          {#if slide}
          {#key slide.id}
            <button
              type="button"
              class="frame"
              class:frame--active={index === activeSlideIndex}
              style={`background:${backgroundColor}; color:${accentColor}; font-family:${fontStack}; --hero-size:${heroFontByType[slide.type] ?? '2rem'}; --hero-scale:${heroScale}; --body-scale:${bodyScale}; --header-scale:${headerScale}; --code-scale:${codeScale};`}
              on:click={() => focusSlide(index)}
              use:captureFrame={{ index }}
            >
              <header class="frame__header">
                <div class="frame__identity">
                  {#if avatarEnabled && avatarSrc}
                    <img src={avatarSrc} alt="Avatar" class="avatar" />
                  {/if}
                  <div>
                    <p class="handle">{personaHandle}</p>
                    {#if personaName?.trim()}
                      <p class="name">{personaName}</p>
                    {/if}
                  </div>
                </div>
              </header>
              <div class="frame__body">
                {#if slide.showHero && slide.parts.hero}
                  <h3 style={`color:${accentColor};`}>
                    {@html renderInline(slide.parts.hero)}
                  </h3>
                {:else if slide.showHero}
                  <h3 style={`color:${accentColor};`}>
                    {slide.fallbackTitle}
                  </h3>
                {/if}
                <div class="markdown">
                  {#if slide.parts.body}
                    {@html renderBody(slide.parts.body)}
                  {:else}
                    <p class="placeholder">Adicione detalhes para este slide.</p>
                  {/if}
                </div>
                {#if slide.type === 'cta'}
                  <span class="cta" style={`background:${accentColor}; color:${backgroundColor}`}>{ctaLabel}</span>
                {/if}
              </div>
            </button>
          {/key}
        {/if}
      {/each}
    </div>

    {#if aiExtras}
      <div class="ai-extras">
        {#if aiExtras.elementos?.length}
          <div class="ai-extras__section">
            <h3>Elementos psicológicos aplicados</h3>
            <ul>
              {#each aiExtras.elementos as item}
                <li>{item}</li>
              {/each}
            </ul>
          </div>
        {/if}

        {#if aiExtras.razoes?.length}
          <div class="ai-extras__section">
            <h3>Por que vai inspirar/viralizar</h3>
            <ul>
              {#each aiExtras.razoes as item}
                <li>{item}</li>
              {/each}
            </ul>
          </div>
        {/if}

        {#if aiExtras.versoesHook?.length}
          <div class="ai-extras__section">
            <h3>Estratégia das 3 versões do hook</h3>
            <ul>
              {#each aiExtras.versoesHook as item}
                <li>{item}</li>
              {/each}
            </ul>
          </div>
        {/if}

        {#if aiExtras.diferenciais?.length}
          <div class="ai-extras__section">
            <h3>Diferenciais únicos</h3>
            <ul>
              {#each aiExtras.diferenciais as item}
                <li>{item}</li>
              {/each}
            </ul>
          </div>
        {/if}
      </div>
    {/if}
  </section>
</main>

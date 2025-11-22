<script>
  import html2canvas from 'html2canvas'
  import { marked } from 'marked'

  marked.setOptions({ breaks: true })

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

  let personaName = ''
  let personaHandle = '@datamundo.cientista'
  let accentColor = '#111111'
  let backgroundColor = '#ffffff'
  let fontStack = 'Space Grotesk, Inter, system-ui, sans-serif'
  let ctaLabel = 'Duplicar template'
  let storyInput = ''
  let generating = false
  let generationError = ''
  let aiExtras = null

  let slidesContent = slideTemplates.map((template) => template.placeholder)
  let activeSlideIndex = 0
  let exporting = false
  let frameRefs = []
  let heroScale = 1
  let bodyScale = 1
  let metaScale = 1
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

  function focusSlide(index) {
    activeSlideIndex = index
  }

  const promptTemplate = `Você é um especialista em criar carrosséis virais para Instagram/LinkedIn seguindo padrões comprovados de alto engajamento.

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
- Campo "hero": frase curta (máx. 5-8 palavras) contextual à história. Não use rótulos genéricos de entonação (ex.: nada de "Momento transformador", "Crise de confiança", "Persistência + Trabalho Duro"). Use uma mini-frase específica da história com verbo/dado que represente a entonação. Não numere.
- Campo "body": texto (máx. 35 palavras), sem repetir o hero.

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

Contexto da história: {HISTORIA}`

  function buildPrompt(story) {
    return promptTemplate.replace('{HISTORIA}', story)
  }

  function formatSlideContent(slideData = {}, template) {
    if (!slideData) return template.placeholder
    const { title, hero, body, versions } = slideData

    if (versions?.length) {
      const hooks = versions.map((text) => text).join('\n')
      return ['Hooks (3 opções)', hooks].filter(Boolean).join('\n')
    }

    const normalizedBody = Array.isArray(body) ? body.join('\n') : body ?? ''
    const heroText = hero || title || template.fallbackTitle

    return [heroText, normalizedBody].filter(Boolean).join('\n')
  }

  function applyAISlides(slidesData = []) {
    slidesContent = slideTemplates.map((template, index) =>
      formatSlideContent(slidesData[index], template)
    )
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

  async function generateWithAI() {
    if (!storyInput.trim()) {
      generationError = 'Descreva a história em um parágrafo curto antes de gerar.'
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
        Coloque um parágrafo curto com a história. A chave vem de <code>.env.local</code> como <code>VITE_OPENAI_API_KEY</code> (não comitar).
      </p>
      <textarea
        rows="4"
        placeholder="Ex: Jovem de 17 anos que vendia brigadeiro na escola virou referência em IA..."
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
        <label class="full range-control">
          <span>Escala título ({heroScale.toFixed(2)}x)</span>
          <input
            type="range"
            min="0.5"
            max="1.25"
            step="0.05"
            bind:value={heroScale}
          />
        </label>
        <label class="full range-control">
          <span>Escala corpo ({bodyScale.toFixed(2)}x)</span>
          <input
            type="range"
            min="0.75"
            max="1.25"
            step="0.05"
            bind:value={bodyScale}
          />
        </label>
        <label class="full range-control">
          <span>Escala meta ({metaScale.toFixed(2)}x)</span>
          <input
            type="range"
            min="0.75"
            max="1.25"
            step="0.05"
            bind:value={metaScale}
          />
        </label>
      </div>
    </div>

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
            <textarea
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
              style={`background:${backgroundColor}; color:${accentColor}; font-family:${fontStack}; --hero-size:${heroFontByType[slide.type] ?? '2rem'}; --hero-scale:${heroScale}; --body-scale:${bodyScale}; --meta-scale:${metaScale};`}
              on:click={() => focusSlide(index)}
              use:captureFrame={{ index }}
            >
              <header class="frame__header">
                <div>
                  <p class="handle">{personaHandle}</p>
                  {#if personaName?.trim()}
                    <p class="name">{personaName}</p>
                  {/if}
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

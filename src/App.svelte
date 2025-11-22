<script>
  import html2canvas from 'html2canvas'
  import { marked } from 'marked'

  marked.setOptions({ breaks: true })

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
                <p class="slide-count">Slide {index + 1}/{totalSlides}</p>
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
  </section>
</main>

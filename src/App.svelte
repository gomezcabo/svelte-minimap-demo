<h2>Minimap generator</h2>
<textarea bind:value={code} placeholder="Paste your code here"></textarea>

{#if blocks.length}
  <button on:click={downloadSVG}>Download SVG</button>
{/if}

<svg bind:this={svgElement} xmlns="http://www.w3.org/2000/svg" width={svgWidth} height={svgHeight} version="1.1">
  {#each blocks as { x, y, width, height, color }}
    <rect x={x} y={y} width={width} height={height} fill={color}/>
  {/each}
</svg>

<script>
  // REPL link:
  // https://svelte.dev/repl/6a615bd496754bc6abc20bef87dd9837?version=3.4.4
  //
  import jslex from './jslex.js'
  import reservedWord from './reservedWord.js'

  const BLOCK_WIDTH = 6
  const BLOCK_HEIGHT = 10

  const tokenTypes = {
    ENTER: 'enter',
    SPACE: 'space',
    NUMBER: 'number',
    STRING: 'string',
    VARIABLE: 'variable',
    HTMLTAG: 'htmltag',
    COMMENT: 'comment',
    SYMBOL: 'symbol'
  }

  const colorForToken = token => {
    switch (reservedWord(token.text)) {
      case 'reserved': return 'darkviolet'
      case 'objects': return 'deepskyblue'
      case 'browser': return 'deepskyblue'
    }

    switch (token.type) {
      case tokenTypes.SYMBOL: return 'darkgray'
      case tokenTypes.NUMBER: return 'coral'
      case tokenTypes.STRING: return 'limegreen'
      case tokenTypes.VARIABLE: return 'gray'
      case tokenTypes.HTMLTAG: return 'coral'
      case tokenTypes.COMMENT: return 'whitesmoke'
      case tokenTypes.SPACE:
      case tokenTypes.ENTER:
      default:
        return null
    }
  }

  const newToken = (type, text) => ({	type, text, length: text.length })

  const lexer = new jslex.jslex({
   'start': {
      '\n': function () {
        return newToken(tokenTypes.ENTER, this.text);
      },
      '[ \t]+': function () {
        return newToken(tokenTypes.SPACE, this.text)
      },
      '[0-9]+': function() {
        return newToken(tokenTypes.NUMBER, this.text)
      },
      '\'.+?\'': function() {
        return newToken(tokenTypes.STRING, this.text)
      },
      '".+?"': function() {
        return newToken(tokenTypes.STRING, this.text)
      },
      '<.+?[(>)|(/\>)|( )|(\n)]': function() {
        return newToken(tokenTypes.HTMLTAG, this.text)
      },
      '/>': function() {
        return newToken(tokenTypes.HTMLTAG, this.text)
      },
      '[a-zA-Z0-9]+': function() {
        return newToken(tokenTypes.VARIABLE, this.text)
      },
      '//.+': function() {
        return newToken(tokenTypes.COMMENT, this.text)
      },
      '(.)': function() {
        return newToken(tokenTypes.SYMBOL, this.text)
      },
    }
  })

  let code = ''

  let blocks = []
  let svgHeight
  let svgWidth

  $: tokens = lexer.collect(code)

  $: if (tokens) {
    let x = 0
    let y = 0

    const enterTokens = tokens.filter(token => token.type === tokenTypes.ENTER).length

    svgHeight = 2 * BLOCK_HEIGHT * (enterTokens + 1) - BLOCK_HEIGHT

    blocks = tokens
      .map(token => {
        const block = {
          ...token,
          x,
          y,
          width: token.length ? BLOCK_WIDTH * token.length: 0,
          height: BLOCK_HEIGHT,
          color: colorForToken(token)
        }

        x += token.type === tokenTypes.ENTER ? -x : block.width
        y += token.type === tokenTypes.ENTER ? 2 * BLOCK_HEIGHT : 0

        return [tokenTypes.ENTER, tokenTypes.SPACE].includes(token.type)
          ? null
          : block
      })
      .filter(token => token !== null)

    svgWidth = Math.max(...blocks.map(block => block.x + block.width))
  }

  let svgElement

  const downloadSVG = () => {
    const svgData = svgElement.outerHTML
    const svgBlob = new Blob([ svgData ], { type: 'image/svg+xml;charset=utf-8' })
    const svgUrl = URL.createObjectURL(svgBlob)

    const downloadLink = document.createElement('a')
    downloadLink.href = svgUrl
    downloadLink.download = 'minimap.svg'

    document.body.appendChild(downloadLink)
    downloadLink.click()
    document.body.removeChild(downloadLink)
  }
</script>

<style>
textarea {
  width: 100%;
  height: 300px;
}
button {
  display: block;
}
</style>


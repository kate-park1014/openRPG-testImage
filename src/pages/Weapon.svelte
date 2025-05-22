<script>
  import { link } from 'svelte-spa-router'
  
  let weaponType = 'sword'
  let style = 'fantasy'
  let material = 'steel'
  let isGenerating = false
  let generatedPrompts = []
  
  const weaponTypes = [
    { value: 'sword', label: '검' },
    { value: 'bow', label: '활' },
    { value: 'staff', label: '지팡이' },
    { value: 'axe', label: '도끼' },
    { value: 'spear', label: '창' },
    { value: 'dagger', label: '단검' }
  ]
  
  const styles = [
    { value: 'fantasy', label: '판타지' },
    { value: 'medieval', label: '중세' },
    { value: 'futuristic', label: '미래형' },
    { value: 'steampunk', label: '스팀펑크' },
    { value: 'magical', label: '마법' }
  ]
  
  const materials = [
    { value: 'steel', label: '강철' },
    { value: 'mithril', label: '미스릴' },
    { value: 'obsidian', label: '흑요석' },
    { value: 'crystal', label: '크리스탈' },
    { value: 'dragon-bone', label: '드래곤 뼈' }
  ]
  
  async function generateWeaponPrompts() {
    isGenerating = true
    try {
      await new Promise(resolve => setTimeout(resolve, 1500))
      
      const prompts = [
        `${style} ${weaponTypes.find(w => w.value === weaponType)?.label} made of ${materials.find(m => m.value === material)?.label}, highly detailed, epic lighting`,
        `Legendary ${weaponTypes.find(w => w.value === weaponType)?.label} forged from ${materials.find(m => m.value === material)?.label}, ${style} style, glowing runes, masterpiece`,
        `Ancient ${weaponTypes.find(w => w.value === weaponType)?.label} of ${materials.find(m => m.value === material)?.label}, ${style} design, intricate details, weapon concept art`,
        `Enchanted ${weaponTypes.find(w => w.value === weaponType)?.label} crafted from ${materials.find(m => m.value === material)?.label}, ${style} aesthetic, magical aura, 4K quality`
      ]
      
      generatedPrompts = prompts
    } finally {
      isGenerating = false
    }
  }
  
  function copyToClipboard(text) {
    navigator.clipboard.writeText(text)
    alert('프롬프트가 클립보드에 복사되었습니다!')
  }
</script>

<div class="min-h-screen bg-gradient-to-br from-red-900 via-orange-900 to-yellow-900 p-8">
  <div class="max-w-4xl mx-auto">
    <!-- 헤더 -->
    <div class="flex items-center justify-between mb-8">
      <h1 class="text-4xl font-bold text-white">⚔️ 무기 프롬프트 생성기</h1>
      <a 
        href="/" 
        use:link 
        class="bg-white/20 hover:bg-white/30 text-white px-6 py-2 rounded-lg transition-colors"
      >
        ← 홈으로
      </a>
    </div>
    
    <!-- 설정 영역 -->
    <div class="bg-white/10 backdrop-blur-md rounded-2xl p-8 mb-8 border border-white/20">
      <h2 class="text-white text-xl font-semibold mb-6">무기 설정</h2>
      
      <div class="grid grid-cols-1 md:grid-cols-3 gap-6">
        <!-- 무기 종류 -->
        <div>
          <label class="block text-white font-semibold mb-2">무기 종류:</label>
          <select bind:value={weaponType} class="w-full px-4 py-2 rounded-lg bg-white/20 text-white border border-white/30 focus:outline-none focus:ring-2 focus:ring-red-400">
            {#each weaponTypes as weapon}
              <option value={weapon.value} class="bg-gray-800">{weapon.label}</option>
            {/each}
          </select>
        </div>
        
        <!-- 스타일 -->
        <div>
          <label class="block text-white font-semibold mb-2">스타일:</label>
          <select bind:value={style} class="w-full px-4 py-2 rounded-lg bg-white/20 text-white border border-white/30 focus:outline-none focus:ring-2 focus:ring-red-400">
            {#each styles as styleOption}
              <option value={styleOption.value} class="bg-gray-800">{styleOption.label}</option>
            {/each}
          </select>
        </div>
        
        <!-- 재료 -->
        <div>
          <label class="block text-white font-semibold mb-2">재료:</label>
          <select bind:value={material} class="w-full px-4 py-2 rounded-lg bg-white/20 text-white border border-white/30 focus:outline-none focus:ring-2 focus:ring-red-400">
            {#each materials as materialOption}
              <option value={materialOption.value} class="bg-gray-800">{materialOption.label}</option>
            {/each}
          </select>
        </div>
      </div>
      
      <div class="mt-6 text-center">
        <button 
          on:click={generateWeaponPrompts}
          disabled={isGenerating}
          class="px-8 py-3 bg-gradient-to-r from-red-500 to-orange-500 hover:from-red-600 hover:to-orange-600 disabled:opacity-50 disabled:cursor-not-allowed text-white font-semibold rounded-lg transition-colors"
        >
          {isGenerating ? '생성 중...' : '프롬프트 생성하기'}
        </button>
      </div>
    </div>
    
    <!-- 결과 영역 -->
    {#if isGenerating}
      <div class="bg-white/10 backdrop-blur-md rounded-2xl p-8 text-center border border-white/20">
        <div class="animate-spin w-12 h-12 border-4 border-red-400 border-t-transparent rounded-full mx-auto mb-4"></div>
        <p class="text-white text-lg">무기 프롬프트를 생성하고 있습니다...</p>
      </div>
    {:else if generatedPrompts.length > 0}
      <div class="bg-white/10 backdrop-blur-md rounded-2xl p-8 border border-white/20">
        <h3 class="text-white text-xl font-semibold mb-6">생성된 무기 프롬프트:</h3>
        <div class="space-y-4">
          {#each generatedPrompts as prompt, index}
            <div class="bg-white/10 rounded-lg p-4 border border-white/20">
              <div class="flex items-start justify-between gap-4">
                <div class="flex-1">
                  <h4 class="text-white font-semibold mb-2">프롬프트 {index + 1}:</h4>
                  <p class="text-gray-200 text-sm leading-relaxed">{prompt}</p>
                </div>
                <button 
                  on:click={() => copyToClipboard(prompt)}
                  class="px-4 py-2 bg-blue-500 hover:bg-blue-600 text-white text-sm rounded-lg transition-colors flex-shrink-0"
                >
                  복사
                </button>
              </div>
            </div>
          {/each}
        </div>
        
        <div class="mt-6 text-center">
          <button 
            on:click={() => generatedPrompts = []}
            class="px-6 py-2 bg-gray-500 hover:bg-gray-600 text-white rounded-lg transition-colors"
          >
            새로 생성
          </button>
        </div>
      </div>
    {/if}
  </div>
</div>

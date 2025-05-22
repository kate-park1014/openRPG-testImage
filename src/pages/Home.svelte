<script>
  import { link, push } from 'svelte-spa-router'
  
  function navigateTo(page) {
    push('/' + page)
  }
</script>

<main class="min-h-screen bg-gradient-main flex items-center justify-center p-8">
  <div class="glass-card rounded-2xl shadow-2xl p-12 text-center max-w-2xl w-full">
    <h1 class="text-5xl font-bold text-white mb-4">
      AI 생성기 도구
    </h1>
    <p class="text-xl text-white mb-12" style="opacity: 0.8;">
      원하는 콘텐츠를 AI로 쉽게 생성해보세요
    </p>
    
    <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
      <!-- Background 생성기 버튼 (a 태그 사용) -->
      <a 
        href="/background" 
        use:link 
        class="btn bg-gradient-blue text-white py-8 px-6"
        style="text-decoration: none; display: flex;"
      >
        <div class="flex-col items-center space-y-4" style="display: flex; flex-direction: column; gap: 1rem; width: 100%;">
          <div style="font-size: 2.5rem;">🎨</div>
          <h3 class="text-xl font-bold">Background, Tag 생성기</h3>
          <p style="font-size: 0.875rem; opacity: 0.9;">멋진 배경 이미지를 AI로 생성</p>
        </div>
      </a>
      
      <!-- 무기 프롬프트 생성기 버튼 (button 사용) -->
      <button 
        on:click={() => navigateTo('weapon')}
        class="btn bg-gradient-red text-white py-8 px-6"
      >
        <div class="flex-col items-center space-y-4" style="display: flex; flex-direction: column; gap: 1rem;">
          <div style="font-size: 2.5rem;">⚔️</div>
          <h3 class="text-xl font-bold">무기 프롬프트 생성기</h3>
          <p style="font-size: 0.875rem; opacity: 0.9;">창의적인 무기 아이디어 생성</p>
        </div>
      </button>
    </div>
    
    <div style="margin-top: 3rem; color: rgba(255,255,255,0.7);">
      <p style="font-size: 0.875rem;">✨ AI 기반 콘텐츠 생성 도구 ✨</p>
    </div>
  </div>
</main>

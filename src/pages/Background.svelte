<script lang="ts">
  import { onMount } from "svelte";
  import OpenAI from "openai";
  import * as XLSX from 'xlsx';
  import { eventCategories } from "../data/categories/categories";

  let isLoading: boolean = false;
  let openai: OpenAI | null = null;
  let response: string = "";
  let eventCount: number = 10; 
  let responseElement: HTMLPreElement;
  let batchSize: number = 10; 
  let currentBatch: number = 0;
  let totalEvents: number = 0; 
  let progressPercent: number = 0; 
  let allResults: any[] = []; 
  let isGenerating: boolean = false; 

  onMount(() => {
    openai = new OpenAI({
      apiKey: import.meta.env.VITE_OPENAI_API_KEY,
      dangerouslyAllowBrowser: true,
    });
  });

  const functions: OpenAI.Chat.ChatCompletionTool[] = [
    {
      type: "function",
      function: {
        name: "generate_event",
        description: "Generate a roguelike game event with associated tags",
        parameters: {
          type: "object",
          properties: {
            tags: { 
              type: "string", 
              description: "string of tags related to the event in english. The first string MUST always be **'forest'**."
            },
            background_image: { 
              type: "string",
              description: "Detailed background description of the event scene for visualization in english"
            },
          },
          required: ["tags", "background"]
        }
      }
    }
  ];

  // 랜덤하게 여러 카테고리 선택하기
  function getRandomCategories(count: number): string[] {
    const shuffled = [...eventCategories].sort(() => 0.5 - Math.random());
    return shuffled.slice(0, count);
  }

  async function generateSingleEvent(category: string): Promise<any> {
    try {
      const completion = await openai?.chat.completions.create({
        model: "gpt-4o-mini",
        messages: [
          {
            role: "system",
            content: [
              {
                type: "text",
                text: `너는 굉장히 뛰어난 게임 시나리오 창작가야. user가 제시한 주제에 대해 로그라이크 게임 이벤트와 관련 태그를 만들어줘. 게임 테마는 "forest"야. 
                1. background_image: 
                - 이벤트 내에서 모험가가 마주하게 될 장면을 시각적으로 상세히 묘사하세요. 
                - flux schnell 모델이 이미지를 생성할 수 있도록 영문 프롬프트로 만들어줘.

                4. tags:
                - 이벤트의 테마, 주제, 요소, 잠재적 게임플레이 영향을 분류하는 약 5개의 관련 tags를 영어로 생성하세요.
                - tags에는 반드시 background_image에서 묘사된 주요 시각적 요소와 분위기를 반영하는 단어들이 포함되어야 합니다.
                - tags에는 처음에 반드시, 테마인 "forest"가 들어가야합니다.
                
                background_image, tags 예:
                { 
                "tags": "forest, wounded warrior, bandage, healing, nature",
                "background_image": "Dim sunlight shines through ancient trees, and beneath them, a wounded warrior lies resting against tree bark. His face is contorted with pain, and the fallen branches and blood-soaked ground around him tell the story of combat. On the ground are his broken sword and a few scattered medicinal herbs.",
                }
                - 이 예시는 단순한 예시일 뿐이며, 실제로는 더 많은 세부사항과 창의적인 요소를 포함해야 합니다.`
              }
            ]
          },
          {
            role: "user",
            content: [
              {
                type: "text",
                text: `"${category}"를 주제로하는 로그라이크 게임 이벤트를 생성해주세요.`
              }
            ]
          }
        ],
        tools: functions,
        tool_choice: { type: "function", function: { name: "generate_event" } },
        temperature: 0.8,
      });

      const message = completion?.choices[0].message;
      
      if (message?.tool_calls && message.tool_calls[0].function.name === "generate_event") {
        return JSON.parse(message.tool_calls[0].function.arguments);
      } else {
        throw new Error("Function call not received");
      }
    } catch (err) {
      console.error("Error generating event:", err);
      return null;
    }
  }

  // 여러 이벤트 생성
  async function generateEvents() {
    if (isLoading || isGenerating || !openai) return;
    
    // 100개 이하는 일반 방식으로 처리
    if (eventCount <= 100) {
      isLoading = true;
      response = "";
      
      const categories = getRandomCategories(eventCount);
      const results: any[] = [];
      
      try {
        for (const category of categories) {
          const result = await generateSingleEvent(category);
          if (result) {
            results.push(result);
          }
        }
        
        const eventData = {};
        eventData["events"] = results;
        
        response = JSON.stringify(eventData, null, 2);
      } catch (error) {
        console.error("Error generating events:", error);
        response = "Error: " + (error instanceof Error ? error.message : String(error));
      } finally {
        isLoading = false;
      }
    } else {
      // 100개 초과는 배치 처리 방식 사용
      startLargeGeneration();
    }
  }
  
  // 대량 이벤트 생성 시작
  async function startLargeGeneration() {
    if (isLoading || isGenerating || !openai) return;
    
    isGenerating = true;
    allResults = [];
    currentBatch = 0;
    totalEvents = eventCount;
    progressPercent = 0;
    
    // 첫 번째 배치 생성 시작
    generateNextBatch();
  }
  
  // 다음 배치 생성
  async function generateNextBatch() {
    if (!isGenerating || !openai) return;
    
    isLoading = true;
    const remainingEvents = totalEvents - currentBatch * batchSize;
    const currentBatchSize = Math.min(batchSize, remainingEvents);
    
    if (currentBatchSize <= 0) {
      // 모든 배치 처리 완료
      finalizeLargeGeneration();
      return;
    }
    
    console.log(`배치 ${currentBatch + 1} 생성 중... (${currentBatchSize}개 이벤트)`);
    
    try {
      const categories = getRandomCategories(currentBatchSize);
      const batchResults: any[] = [];
      
      for (let i = 0; i < categories.length; i++) {
        const category = categories[i];
        const result = await generateSingleEvent(category);
        if (result) {
          batchResults.push(result);
        }
        
        // 진행률 업데이트 (배치 내에서도 업데이트)
        const processed = currentBatch * batchSize + i + 1;
        progressPercent = Math.round((processed / totalEvents) * 100);
        
        // 매 이벤트 생성 후 화면 업데이트
        if (i % 2 === 0 || i === categories.length - 1) {
          allResults = [...allResults, ...batchResults.slice(i-1 >= 0 ? i-1 : 0, i+1)];
          const eventData = {};
          eventData["events"] = allResults;
          response = JSON.stringify(eventData, null, 2);
        }
      }
      
      // 배치의 모든 결과 저장
      allResults = [...allResults.slice(0, currentBatch * batchSize), ...batchResults];
      
      // 임시 결과 표시
      const eventData = {};
      eventData["events"] = allResults;
      response = JSON.stringify(eventData, null, 2);
      
      // 다음 배치로 이동
      currentBatch++;
      isLoading = false;
      
      // 다음 배치 처리 (약간의 지연 추가)
      setTimeout(generateNextBatch, 500);
    } catch (error) {
      console.error("Error generating batch:", error);
      isLoading = false;
      isGenerating = false;
    }
  }
  
  // 대량 생성 완료
  function finalizeLargeGeneration() {
    isLoading = false;
    isGenerating = false;
    
    // 최종 결과 포맷팅
    const eventData = {};
    eventData["events"] = allResults;
    response = JSON.stringify(eventData, null, 2);
    
    console.log(`총 ${allResults.length}개의 이벤트 생성 완료`);
  }
  
  // 배치 크기 변경 핸들러
  function handleBatchSizeChange(e: Event) {
    const target = e.target as HTMLSelectElement;
    batchSize = parseInt(target.value);
    console.log(`배치 크기가 ${batchSize}개로 변경되었습니다.`);
  }
  
  // 이벤트 개수 변경 핸들러
  function handleCountChange(e: Event) {
    const target = e.target as HTMLSelectElement;
    eventCount = parseInt(target.value);
    console.log(`이벤트 생성 개수가 ${eventCount}개로 변경되었습니다.`);
  }
  
  // 엑셀 파일 다운로드 함수
  function downloadExcel() {
    if (!response) return;
    
    try {
      // 현재 이벤트 데이터 가져오기
      const eventData = JSON.parse(response);
      const events = eventData.events || [];
      
      if (events.length === 0) {
        alert('다운로드할 이벤트 데이터가 없습니다.');
        return;
      }
      
      // 엑셀 워크시트 데이터 준비
      const worksheetData = [];
      
      // 헤더 추가
      worksheetData.push(['이벤트 내용', '태그']);
      
      // 각 이벤트 데이터 추가
      events.forEach(event => {
        worksheetData.push([
          event.content,
          event.tags.join(', ')
        ]);
      });
      
      // 워크시트 생성
      const worksheet = XLSX.utils.aoa_to_sheet(worksheetData);
      
      // 열 너비 자동 조정을 위한 설정
      const maxWidths = [0, 0]; // 각 열의 최대 너비
      
      worksheetData.forEach(row => {
        row.forEach((cell, i) => {
          const cellLength = String(cell).length;
          if (cellLength > maxWidths[i]) {
            maxWidths[i] = cellLength;
          }
        });
      });
      
      // 워크시트 열 너비 설정
      worksheet['!cols'] = [
        { wch: Math.min(maxWidths[0], 100) }, // 이벤트 내용 (최대 100자 제한)
        { wch: Math.min(maxWidths[1], 80) }   // 태그 (최대 80자 제한)
      ];
      
      // 워크북 생성 및 워크시트 추가
      const workbook = XLSX.utils.book_new();
      XLSX.utils.book_append_sheet(workbook, worksheet, '로그라이크 이벤트');
      
      // 엑셀 파일 생성 및 다운로드
      XLSX.writeFile(workbook, '로그라이크_이벤트.xlsx');
      
      console.log(`${events.length}개의 이벤트가 엑셀 파일로 저장되었습니다.`);
    } catch (error) {
      console.error('엑셀 파일 생성 중 오류 발생:', error);
      alert('엑셀 파일 생성 중 오류가 발생했습니다.');
    }
  }
  
  // 별도 JSON 형식 다운로드 함수 (ID 목적)
  function downloadEventWithId() {
    if (!response) return;
    
    try {
      // 현재 이벤트 데이터 가져오기
      const eventData = JSON.parse(response);
      const events = eventData.events || [];
      
      if (events.length === 0) {
        alert('다운로드할 이벤트 데이터가 없습니다.');
        return;
      }
      
      // ID 추가하기
      const eventsWithId = events.map((event, index) => ({
        id: index + 1,
        ...event
      }));
      
      // 새로운 JSON 객체 생성
      const idJsonData = {
        events: eventsWithId
      };
      
      // JSON 문자열로 변환
      const jsonString = JSON.stringify(idJsonData, null, 2);
      
      // Blob 객체 생성
      const blob = new Blob([jsonString], { type: 'application/json' });
      
      // 다운로드 링크 생성
      const url = URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      a.download = 'db_with_id.json';
      
      // 링크 클릭하여 다운로드 시작
      document.body.appendChild(a);
      a.click();
      
      // 정리
      setTimeout(() => {
        document.body.removeChild(a);
        URL.revokeObjectURL(url);
      }, 100);
    } catch (error) {
      console.error('ID 포함 JSON 파일 다운로드 중 오류 발생:', error);
      alert('파일 다운로드 중 오류가 발생했습니다.');
    }
  }


</script>

<main class="max-w-2xl mx-auto p-5">
  <h1 class="text-3xl font-bold mb-6 text-center">
    background, tags 생성기
  </h1>
  <div class="flex flex-col gap-4 mb-6">
    <div class="flex justify-between items-center">
      <label for="eventCount" class="text-lg font-medium">생성할 이벤트 개수:</label>
      <select 
        id="eventCount" 
        on:change={handleCountChange} 
        class="p-2 border rounded-lg"
        disabled={isLoading || isGenerating}
      > 
        <option value="1">1개</option>
        <option value="5">5개</option>
        <option value="10" selected>10개</option>
        <option value="20">20개</option>
        <option value="50">50개</option>
        <option value="100">100개</option>
        <option value="200">200개</option>
        <option value="500">500개</option>
        <option value="1000">1,000개</option>
        <option value="5000">5,000개</option>
        <option value="10000">10,000개</option>
      </select>
    </div>
    
    <div class="flex justify-between items-center">
      <label for="batchSize" class="text-lg font-medium">배치 크기 (대량 생성 시):</label>
      <select 
        id="batchSize" 
        on:change={handleBatchSizeChange} 
        class="p-2 border rounded-lg"
        disabled={isLoading || isGenerating}
      >
        <option value="5">5개씩</option>
        <option value="10" selected>10개씩</option>
        <option value="20">20개씩</option>
        <option value="50">50개씩</option>
      </select>
    </div>
    
    <button
      on:click={generateEvents}
      disabled={isLoading || isGenerating}
      class="w-full p-4 bg-blue-600 text-white rounded-lg hover:bg-blue-700 disabled:bg-gray-400 disabled:cursor-not-allowed text-lg font-medium"
    >
      {#if isLoading || isGenerating}
        {#if progressPercent > 0}
          이벤트 생성 중... ({progressPercent}%)
        {:else}
          이벤트 생성 중...
        {/if}
      {:else}
        이벤트 생성하기
      {/if}
    </button>
    
    {#if isGenerating || isLoading}
      <div class="w-full bg-gray-200 rounded-full h-4 mt-2">
        <div class="bg-blue-600 h-4 rounded-full" style="width: {progressPercent}%"></div>
      </div>
    {/if}
  </div>

  {#if response}
    <div class="mt-6">
      <h2 class="text-xl font-semibold mb-3">생성된 이벤트 ({allResults.length > 0 ? allResults.length : JSON.parse(response).events.length}개)</h2>
      <div class="flex justify-end mb-2 gap-2">
        <button 
          on:click={() => {
            if (responseElement) {
              navigator.clipboard.writeText(responseElement.textContent || '');
            }
          }}
          class="px-3 py-1 bg-gray-200 text-gray-800 rounded hover:bg-gray-300"
        >
          복사하기
        </button>
        <button 
          on:click={downloadEventWithId}
          class="px-3 py-1 bg-blue-500 text-white rounded hover:bg-blue-600"
        >
          db.json 다운로드
        </button>
        <button 
          on:click={downloadExcel}
          class="px-3 py-1 bg-green-500 text-white rounded hover:bg-green-600"
        >
          Excel 다운로드
        </button>
      </div>
      <pre
        bind:this={responseElement}
        class="bg-gray-100 p-4 rounded-lg whitespace-pre-wrap break-words max-h-96 overflow-y-auto"
      >{response}</pre>
    </div>
  {/if}
</main>
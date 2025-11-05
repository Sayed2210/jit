<script setup lang="ts">
import {
  ref,
  reactive,
  computed,
  watch,
  onMounted,
  onBeforeUnmount,
  onUnmounted,
} from "vue";

// ============= STATE MANAGEMENT =============
const activeMode = ref("compiled");
const isRunning = ref(false);
const progress = ref(0);
const executionLog = ref([]);
const currentLine = ref(-1);
const hotspots = ref({});
const totalTime = ref(0);
const compiledCode = ref([]);

let intervalId = null;
let startTime = 0;

// ============= DATA DEFINITIONS =============
const sourceCode = [
  "function calculateSum() {",
  "  let sum = 0;",
  "  for (let i = 0; i < 5; i++) {",
  "    sum += i;",
  "  }",
  "  return sum;",
  "}",
];

const machineCode = [
  "MOV R0, #0      ; sum = 0",
  "MOV R1, #0      ; i = 0",
  "LOOP:",
  "CMP R1, #5      ; compare i < 5",
  "BGE END         ; branch if >=",
  "ADD R0, R0, R1  ; sum += i",
  "ADD R1, R1, #1  ; i++",
  "B LOOP          ; jump to loop",
  "END:",
  "RET             ; return sum",
];

const modes = [
  {
    id: "compiled",
    title: "Compiled (AOT)",
    tagline: "Fast execution, slow build",
    icon: "🔧",
    color: "blue",
    description: `
Compiled languages translate your entire program into machine code
**before** it ever runs. This happens through a compiler (like GCC or Rustc)
that reads your source code, checks for syntax or type errors, and
converts it into a binary executable (.exe, .out, etc.).  
Once compiled, the CPU executes native instructions **directly**, so the runtime
performance is extremely fast. However, you must wait for the
compilation process every time you build your project.  
Compiled programs are best when you care about **speed, efficiency, and reliability**.
Examples: C, C++, Rust, Go, Swift.`,
    speed: "⚡⚡⚡ Very Fast",
    startup: "🐌 Slow (requires full compilation)",
    bestFor:
      "Operating systems, Games, Native Apps, Performance-critical systems",
  },
  {
    id: "interpreted",
    title: "Interpreted",
    tagline: "Instant start, slower execution",
    icon: "▶️",
    color: "green",
    description: `
Interpreted languages execute code **line by line** without producing
a separate binary. The interpreter reads, analyzes, and runs each
instruction directly during runtime.  
This means your code can start running immediately—no need to compile—
but every line must be processed repeatedly each time the program runs,
making it slower overall.  
Interpreted languages are flexible, dynamic, and great for **rapid prototyping**
or scripting, where development speed matters more than raw performance.  
Examples: Python, Ruby, PHP, JavaScript (in its pure interpreted form).`,
    speed: "🐌 Slow",
    startup: "⚡⚡⚡ Instant (no compilation needed)",
    bestFor: "Scripts, Prototypes, Education, Small-scale Automation",
  },
  {
    id: "jit",
    title: "JIT (Just-In-Time) Compiled",
    tagline: "Hybrid execution — interpreted + compiled",
    icon: "⚡",
    color: "purple",
    description: `
JIT (Just-In-Time) compilation combines the best of both worlds:
it starts executing code immediately (like an interpreter), and while
the program runs, it analyzes which parts (called *hotspots*) are used most often.
Those frequently used sections are then compiled **on the fly** into machine code
for faster performance.  
This adaptive optimization approach allows programs to start fast and
gradually reach near-native performance as they run longer.  
It’s how modern platforms achieve both responsiveness and speed.  
Examples: Java (HotSpot JVM), C# (.NET CLR), JavaScript (V8, SpiderMonkey).`,
    speed: "⚡⚡ Fast (after warm-up)",
    startup: "⚡⚡ Fast (minimal precompilation)",
    bestFor:
      "Web browsers, Virtual Machines, Enterprise and Cross-platform apps",
  },
];

// ============= COMPUTED PROPERTIES =============
const currentModeInfo = computed(() => {
  return modes.find((m) => m.id === activeMode.value) || modes[0];
});

const displayCode = computed(() => {
  return showMachineCode.value ? machineCode : sourceCode;
});

const showMachineCode = computed(() => {
  return activeMode.value === "compiled" && compiledCode.value.length > 0;
});

// ============= METHODS =============
const addLog = (message, type = "default") => {
  executionLog.value.push({
    message,
    type,
    time: Date.now(),
  });
  // Keep only last 12 logs
  if (executionLog.value.length > 12) {
    executionLog.value.shift();
  }
};

const formatTime = (timestamp) => {
  const date = new Date(timestamp);
  return date.toLocaleTimeString("en-US", {
    hour12: false,
    hour: "2-digit",
    minute: "2-digit",
    second: "2-digit",
  });
};

const selectMode = (modeId) => {
  if (!isRunning.value) {
    activeMode.value = modeId;
    handleReset();
  }
};

const handleStart = () => {
  if (activeMode.value === "compiled") {
    // Simulate compilation phase
    addLog("⚙️ Starting compilation process...", "info");
    addLog("📖 Parsing source code...", "info");

    setTimeout(() => {
      addLog("🔍 Performing lexical analysis...", "info");
    }, 500);

    setTimeout(() => {
      addLog("🌳 Building abstract syntax tree...", "info");
    }, 1000);

    setTimeout(() => {
      compiledCode.value = [...machineCode];
      addLog("✅ Compilation complete! Binary ready.", "success");
      addLog("🚀 Starting execution...", "info");
      startExecution();
    }, 1500);
  } else {
    addLog(`🚀 Starting ${currentModeInfo.value.title} execution...`, "info");
    if (activeMode.value === "jit") {
      addLog("📊 JIT profiler initialized - monitoring hot paths...", "info");
    }
    startExecution();
  }
};

const startExecution = () => {
  isRunning.value = true;
  startTime = Date.now();

  const speed =
    activeMode.value === "compiled"
      ? 150
      : activeMode.value === "jit"
        ? 200
        : 300;

  let lineIndex = 0;
  const codeArray = displayCode.value;

  intervalId = setInterval(() => {
    // Update progress
    progress.value = Math.min(progress.value + 100 / codeArray.length, 100);
    totalTime.value = Date.now() - startTime;

    if (lineIndex < codeArray.length) {
      currentLine.value = lineIndex;

      // Mode-specific execution logic
      if (activeMode.value === "interpreted") {
        addLog(`📝 Parsing line ${lineIndex + 1}: ${codeArray[lineIndex]}`);
        addLog(`▶️ Executing line ${lineIndex + 1}...`);
      } else if (activeMode.value === "compiled") {
        addLog(`⚡ Executing: ${codeArray[lineIndex]}`);
      } else if (activeMode.value === "jit") {
        // Track hotspots for loop lines
        if (lineIndex >= 2 && lineIndex <= 4) {
          hotspots.value[lineIndex] = (hotspots.value[lineIndex] || 0) + 1;

          if (hotspots.value[lineIndex] === 2) {
            addLog(
              `🔥 Hot spot detected at line ${lineIndex + 1}! Compiling to native code...`,
              "warning"
            );
            setTimeout(() => {
              addLog(
                `✅ Line ${lineIndex + 1} compiled! Now executing at native speed`,
                "success"
              );
            }, 100);
          } else if (hotspots.value[lineIndex] > 2) {
            addLog(`⚡ Executing optimized code: ${codeArray[lineIndex]}`);
          } else {
            addLog(
              `🔍 Interpreting line ${lineIndex + 1}: ${codeArray[lineIndex]}`
            );
          }
        } else {
          addLog(
            `🔍 Interpreting line ${lineIndex + 1}: ${codeArray[lineIndex]}`
          );
        }
      }

      lineIndex++;
    } else {
      // Execution complete
      clearInterval(intervalId);
      isRunning.value = false;
      progress.value = 100;
      addLog("🎉 Execution completed successfully!", "success");
      addLog(`📊 Total execution time: ${totalTime.value}ms`, "info");
    }
  }, speed);
};

const handleReset = () => {
  if (intervalId) {
    clearInterval(intervalId);
  }
  isRunning.value = false;
  progress.value = 0;
  compiledCode.value = [];
  executionLog.value = [];
  currentLine.value = -1;
  hotspots.value = {};
  totalTime.value = 0;
};

// ============= WATCHERS =============
watch(activeMode, () => {
  handleReset();
});

// ============= LIFECYCLE HOOKS =============
onUnmounted(() => {
  if (intervalId) {
    clearInterval(intervalId);
  }
});
</script>

<template>
  <div
    class="min-h-screen bg-gradient-to-br from-gray-900 via-gray-800 to-gray-900 text-white p-4 md:p-8"
  >
    <div class="max-w-7xl mx-auto">
      <!-- Header Section -->
      <div class="text-center mb-8">
        <h1 class="text-3xl md:text-4xl font-bold mb-2">
          Code Execution Visualizer
        </h1>
        <p class="text-gray-400">
          Interactive demonstration of Compiled, Interpreted, and JIT execution
        </p>
        <p class="text-sm text-gray-500 mt-2">
          Built with Vue.js 3 Composition API
        </p>
      </div>

      <!-- Mode Selection Tabs -->
      <div class="grid grid-cols-1 md:grid-cols-3 gap-4 mb-8">
        <button
          v-for="mode in modes"
          :key="mode.id"
          @click="selectMode(mode.id)"
          :class="[
            'p-4 rounded-lg border-2 transition-all duration-300 transform hover:scale-105',
            activeMode === mode.id
              ? `border-${mode.color}-500 bg-gray-800 shadow-lg`
              : 'border-gray-700 bg-gray-800/50 hover:border-gray-600',
          ]"
        >
          <div class="flex items-center justify-center gap-2 mb-2">
            <span v-html="mode.icon"></span>
            <span class="font-semibold">{{ mode.title }}</span>
          </div>
          <div class="text-xs text-gray-400">{{ mode.tagline }}</div>
        </button>
      </div>

      <!-- Mode Information Card -->
      <transition name="fade">
        <div
          :class="[
            'border-2 rounded-lg p-6 mb-8 transition-all duration-300',
            `bg-${currentModeInfo.color}-500/10 border-${currentModeInfo.color}-500`,
          ]"
        >
          <div class="flex flex-col md:flex-row items-start gap-4">
            <div :class="['p-3 rounded-lg', `bg-${currentModeInfo.color}-500`]">
              <span v-html="currentModeInfo.icon" class="text-2xl"></span>
            </div>
            <div class="flex-1">
              <h2 class="text-xl md:text-2xl font-bold mb-2">
                {{ currentModeInfo.title }}
              </h2>
              <p class="text-gray-300 mb-4">
                {{ currentModeInfo.description }}
              </p>
              <div class="grid grid-cols-1 md:grid-cols-3 gap-4">
                <div>
                  <span class="text-gray-400 text-sm">Execution Speed:</span>
                  <div class="font-semibold">{{ currentModeInfo.speed }}</div>
                </div>
                <div>
                  <span class="text-gray-400 text-sm">Startup Time:</span>
                  <div class="font-semibold">{{ currentModeInfo.startup }}</div>
                </div>
                <div>
                  <span class="text-gray-400 text-sm">Best For:</span>
                  <div class="font-semibold text-sm">
                    {{ currentModeInfo.bestFor }}
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </transition>

      <!-- Control Buttons -->
      <div class="flex flex-wrap gap-4 mb-8 justify-center">
        <button
          @click="handleStart"
          :disabled="isRunning || progress === 100"
          :class="[
            'flex items-center gap-2 px-6 py-3 rounded-lg font-semibold transition-all duration-300 transform hover:scale-105',
            isRunning || progress === 100
              ? 'bg-gray-700 cursor-not-allowed opacity-50'
              : 'bg-green-600 hover:bg-green-700 shadow-lg',
          ]"
        >
          <span>▶</span>
          {{ activeMode === "compiled" ? "Compile & Run" : "Run" }}
        </button>
        <button
          @click="handleReset"
          class="flex items-center gap-2 px-6 py-3 bg-gray-700 hover:bg-gray-600 rounded-lg font-semibold transition-all duration-300 transform hover:scale-105"
        >
          <span>↻</span>
          Reset
        </button>
      </div>

      <!-- Progress Bar -->
      <div class="mb-8">
        <div class="flex justify-between mb-2">
          <span class="text-sm text-gray-400">Execution Progress</span>
          <span class="text-sm text-gray-400">{{ Math.round(progress) }}%</span>
        </div>
        <div class="w-full bg-gray-700 rounded-full h-4 overflow-hidden">
          <div
            :class="`h-full bg-${currentModeInfo.color}-500 transition-all duration-300`"
            :style="{ width: progress + '%' }"
          ></div>
        </div>
        <div class="flex justify-between mt-2">
          <span class="text-xs text-gray-500 flex items-center gap-1">
            ⏱ Total Time: {{ totalTime }}ms
          </span>
          <span
            v-if="activeMode === 'jit' && Object.keys(hotspots).length > 0"
            class="text-xs text-orange-400"
          >
            🔥 Hot spots compiled:
            {{ Object.keys(hotspots).filter((k) => hotspots[k] >= 2).length }}
          </span>
        </div>
      </div>

      <!-- Main Content Grid -->
      <div class="grid grid-cols-1 lg:grid-cols-2 gap-8 mb-8">
        <!-- Source Code Panel -->
        <div class="bg-gray-800 rounded-lg p-4 md:p-6 border border-gray-700">
          <h3
            class="text-lg md:text-xl font-semibold mb-4 flex items-center gap-2"
          >
            <span>📝</span>
            {{ showMachineCode ? "Machine Code (Compiled)" : "Source Code" }}
          </h3>
          <div class="font-mono text-xs md:text-sm space-y-1 overflow-x-auto">
            <div
              v-for="(line, idx) in displayCode"
              :key="idx"
              :class="[
                'p-2 rounded transition-all duration-300',
                currentLine === idx
                  ? 'bg-yellow-500/20 border-l-4 border-yellow-500 transform scale-105'
                  : '',
                hotspots[idx] >= 2 ? 'bg-orange-500/10' : '',
              ]"
            >
              <span class="text-gray-500 mr-2 md:mr-4 select-none">{{
                idx + 1
              }}</span>
              <span
                :class="
                  hotspots[idx] >= 2 ? 'text-orange-400' : 'text-gray-300'
                "
              >
                {{ line }}
                <span
                  v-if="hotspots[idx] >= 2"
                  class="ml-2 text-xs bg-orange-500/20 px-2 py-1 rounded"
                >
                  🔥 COMPILED
                </span>
              </span>
            </div>
          </div>
        </div>

        <!-- Execution Log Panel -->
        <div class="bg-gray-800 rounded-lg p-4 md:p-6 border border-gray-700">
          <h3
            class="text-lg md:text-xl font-semibold mb-4 flex items-center gap-2"
          >
            <span>📊</span>
            Execution Log
          </h3>
          <div
            class="space-y-2 font-mono text-xs md:text-sm h-64 md:h-80 overflow-y-auto custom-scrollbar"
          >
            <div
              v-if="executionLog.length === 0"
              class="text-gray-500 text-center py-8"
            >
              Click "Run" to start execution...
            </div>
            <transition-group name="slide-up" v-else>
              <div
                v-for="(log, idx) in executionLog"
                :key="log.time + idx"
                :class="[
                  'p-2 rounded transition-all duration-300',
                  log.type === 'success'
                    ? 'bg-green-500/10 text-green-400 border-l-2 border-green-500'
                    : log.type === 'warning'
                      ? 'bg-orange-500/10 text-orange-400 border-l-2 border-orange-500'
                      : log.type === 'info'
                        ? 'bg-blue-500/10 text-blue-400 border-l-2 border-blue-500'
                        : 'text-gray-300',
                ]"
              >
                <span class="text-gray-500 text-xs mr-2">{{
                  formatTime(log.time)
                }}</span>
                {{ log.message }}
              </div>
            </transition-group>
          </div>
        </div>
      </div>

      <!-- Completion Stats -->
      <transition name="fade">
        <div
          v-if="progress === 100"
          class="bg-gradient-to-r from-green-900/30 to-blue-900/30 rounded-lg p-6 border border-green-500/50"
        >
          <h3
            class="text-xl md:text-2xl font-semibold mb-4 flex items-center gap-2"
          >
            <span>🎉</span>
            Execution Complete!
          </h3>
          <div class="grid grid-cols-2 md:grid-cols-4 gap-4">
            <div class="bg-gray-800/50 p-4 rounded-lg">
              <div class="text-gray-400 text-sm mb-1">Total Time</div>
              <div class="text-2xl font-bold text-green-400">
                {{ totalTime }}ms
              </div>
            </div>
            <div class="bg-gray-800/50 p-4 rounded-lg">
              <div class="text-gray-400 text-sm mb-1">Execution Mode</div>
              <div class="text-2xl font-bold">{{ currentModeInfo.title }}</div>
            </div>
            <div class="bg-gray-800/50 p-4 rounded-lg">
              <div class="text-gray-400 text-sm mb-1">Lines Executed</div>
              <div class="text-2xl font-bold text-blue-400">
                {{ displayCode.length }}
              </div>
            </div>
            <div class="bg-gray-800/50 p-4 rounded-lg">
              <div class="text-gray-400 text-sm mb-1">Performance</div>
              <div class="text-2xl font-bold text-purple-400">
                {{
                  activeMode === "compiled"
                    ? "⚡⚡⚡"
                    : activeMode === "jit"
                      ? "⚡⚡"
                      : "⚡"
                }}
              </div>
            </div>
          </div>
        </div>
      </transition>

      <!-- Documentation Section -->
      <!-- <div class="mt-8 bg-gray-800 rounded-lg p-6 border border-gray-700">
        <h3
          class="text-xl md:text-2xl font-semibold mb-4 flex items-center gap-2"
        >
          <span>📚</span>
          How This Works - Technical Documentation
        </h3>

        <div class="space-y-6 text-gray-300">
          <div>
            <h4 class="text-lg font-semibold text-white mb-2">
              🔧 Vue.js 3 Implementation
            </h4>
            <ul class="list-disc list-inside space-y-1 text-sm">
              <li>
                <strong>Composition API:</strong> Uses reactive refs and
                computed properties for state management
              </li>
              <li>
                <strong>Watchers:</strong> Monitors activeMode changes and
                triggers reset automatically
              </li>
              <li>
                <strong>Lifecycle Hooks:</strong> onMounted and onUnmounted for
                interval cleanup
              </li>
              <li>
                <strong>Transitions:</strong> Built-in Vue transitions for
                smooth UI updates
              </li>
            </ul>
          </div>

          <div>
            <h4 class="text-lg font-semibold text-white mb-2">
              ⚙️ Compiled Mode Simulation
            </h4>
            <ul class="list-disc list-inside space-y-1 text-sm">
              <li>Simulates AOT (Ahead-of-Time) compilation with 1.5s delay</li>
              <li>
                Translates source code to pseudo machine code (assembly-like
                instructions)
              </li>
              <li>Executes at 150ms per instruction (fastest mode)</li>
              <li>Demonstrates: C, C++, Rust, Go compilation process</li>
            </ul>
          </div>

          <div>
            <h4 class="text-lg font-semibold text-white mb-2">
              🐌 Interpreted Mode Simulation
            </h4>
            <ul class="list-disc list-inside space-y-1 text-sm">
              <li>Line-by-line parsing and execution (no pre-compilation)</li>
              <li>Runs at 300ms per line (slowest mode)</li>
              <li>Shows both parsing and execution steps in real-time</li>
              <li>
                Demonstrates: Python, Ruby, PHP traditional interpretation
              </li>
            </ul>
          </div>

          <div>
            <h4 class="text-lg font-semibold text-white mb-2">
              🔥 JIT Compilation Simulation
            </h4>
            <ul class="list-disc list-inside space-y-1 text-sm">
              <li>Starts in interpreted mode (instant startup)</li>
              <li>Tracks hotspots: code executed 2+ times gets flagged</li>
              <li>Automatically compiles hot code paths to "machine code"</li>
              <li>Compiled sections execute faster on subsequent runs</li>
              <li>Runs at 200ms per line (medium speed with optimization)</li>
              <li>Demonstrates: Java HotSpot, JavaScript V8, .NET CLR</li>
            </ul>
          </div>

          <div>
            <h4 class="text-lg font-semibold text-white mb-2">
              🎯 Key Features
            </h4>
            <ul class="list-disc list-inside space-y-1 text-sm">
              <li>
                <strong>Real-time Highlighting:</strong> currentLine tracks
                execution position
              </li>
              <li>
                <strong>Hotspot Detection:</strong> Object tracks line execution
                frequency
              </li>
              <li>
                <strong>Performance Metrics:</strong> Accurate timing with
                different speeds per mode
              </li>
              <li>
                <strong>Responsive Design:</strong> Tailwind CSS with
                mobile-first approach
              </li>
              <li>
                <strong>Clean Architecture:</strong> Separated concerns with
                computed properties
              </li>
            </ul>
          </div>

          <div>
            <h4 class="text-lg font-semibold text-white mb-2">
              📊 Performance Comparison
            </h4>
            <div class="bg-gray-900 p-4 rounded mt-2 text-xs md:text-sm">
              <div>
                Compiled: 150ms/instruction → Total: ~1500ms + 1500ms
                compilation = 3000ms
              </div>
              <div>
                JIT: 200ms/line → Total: ~1400ms (no compilation delay, runtime
                optimization)
              </div>
              <div>
                Interpreted: 300ms/line → Total: ~2100ms (slowest, but instant
                start)
              </div>
            </div>
          </div>

          <div>
            <h4 class="text-lg font-semibold text-white mb-2">💡 Try This</h4>
            <ul class="list-disc list-inside space-y-1 text-sm">
              <li>Run all three modes and compare total execution times</li>
              <li>Watch how JIT detects the loop (lines 3-5) as a hotspot</li>
              <li>
                Notice the compilation delay in Compiled mode vs instant start
                in others
              </li>
              <li>
                Observe the log messages showing different execution strategies
              </li>
            </ul>
          </div>
        </div>
      </div> -->

      <!-- Footer -->
      <div class="mt-8 text-center text-sm text-gray-500">
        <p>Built with Vue.js 3 (Front End Team)</p>
        <!-- <p class="mt-2">Educational visualization of code execution methods</p> -->
      </div>
    </div>
  </div>
</template>

<style scoped>
.custom-scrollbar::-webkit-scrollbar {
  width: 8px;
}
.custom-scrollbar::-webkit-scrollbar-track {
  background: #374151;
  border-radius: 4px;
}
.custom-scrollbar::-webkit-scrollbar-thumb {
  background: #6b7280;
  border-radius: 4px;
}
.custom-scrollbar::-webkit-scrollbar-thumb:hover {
  background: #9ca3af;
}
</style>

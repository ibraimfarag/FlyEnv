<template>
  <div class="llm-checker-wrap h-full overflow-auto p-4">
    <div class="flex items-center justify-between mb-3">
      <div class="text-lg font-semibold">LLM Checker</div>
      <el-button :loading="state.fetching" @click="refresh">Recheck</el-button>
    </div>

    <el-alert
      v-if="state.error"
      :title="state.error"
      type="warning"
      :closable="false"
      class="mb-3"
    />

    <el-alert v-if="!ollamaServiceRunning" type="error" :closable="false" class="mb-3">
      <template #title>
        <span>
          You should go to run Ollama.
          <el-button link type="primary" @click="goToOllamaServiceTab">
            Click here to go to Ollama tab
          </el-button>
        </span>
      </template>
    </el-alert>

    <el-card class="mb-3" shadow="never">
      <template #header>
        <div class="font-medium">PC Report Summary</div>
      </template>
      <div class="grid grid-cols-1 md:grid-cols-2 gap-y-2 gap-x-6 text-sm">
        <div v-for="item in summaryItems" :key="item.key" class="summary-row">
          <span class="summary-key">{{ item.key }}:</span>
          <span class="summary-value">{{ item.value }}</span>
        </div>
      </div>
      <div class="mt-3 text-xs text-gray-500">
        {{ machineHint }}
      </div>

      <el-divider class="!my-4" />

      <div class="text-sm font-medium mb-2">Graphics Cards</div>
      <el-tabs v-model="state.gpuTab" type="card" class="gpu-tabs">
        <el-tab-pane
          v-for="(item, idx) in gpuRows"
          :key="`${idx}-${item.cardName}`"
          :name="`${idx}`"
          :label="`GPU ${idx + 1}: ${item.cardName}`"
        >
          <el-table :data="[item]" :border="false" size="small" style="width: 100%">
            <el-table-column prop="vendor" label="Card Name" min-width="140" />
            <el-table-column prop="model" label="Model" min-width="180" />
            <el-table-column prop="vram" label="RAM" width="140" />
            <el-table-column prop="driverVersion" label="Driver" min-width="120" />
            <el-table-column prop="resolution" label="Resolution" width="130" />
            <el-table-column prop="processor" label="Video Processor" min-width="180" />
          </el-table>
        </el-tab-pane>
        <el-tab-pane v-if="!gpuRows.length" name="empty" label="GPU 1: Unknown">
          <div class="text-xs text-gray-500 py-2">No graphics card information found.</div>
        </el-tab-pane>
      </el-tabs>
    </el-card>

    <el-card shadow="never">
      <template #header>
        <div class="font-medium">Model Fit & Suggestions</div>
      </template>

      <div class="mb-3 text-xs text-gray-500">
        Device performance profile: <b>{{ performanceTier.label }}</b> · GPU VRAM:
        <b>{{ performanceTier.maxVramText }}</b> · System RAM: <b>{{ performanceTier.systemRamText }}</b>
      </div>

      <div class="mb-3 flex flex-wrap gap-3 items-center">
        <div class="flex items-center gap-2">
          <span class="text-xs text-gray-500">Category</span>
          <el-select v-model="state.modelCategory" style="width: 220px" size="small">
            <el-option
              v-for="item in categoryOptions"
              :key="item.value"
              :label="item.label"
              :value="item.value"
            />
          </el-select>
        </div>
        <div class="flex items-center gap-2">
          <span class="text-xs text-gray-500">Display</span>
          <el-radio-group v-model="state.modelDisplay" size="small">
            <el-radio-button value="table">Table</el-radio-button>
            <el-radio-button value="cards">Cards</el-radio-button>
          </el-radio-group>
        </div>
      </div>

      <div class="mb-3 flex flex-wrap gap-2 items-center">
        <span class="text-xs text-gray-500">Models Family</span>
        <el-button
          size="small"
          :type="state.modelFamily === 'all' ? 'primary' : 'default'"
          @click="state.modelFamily = 'all'"
        >
          All
        </el-button>
        <el-button
          v-for="family in familyOptions"
          :key="`family-${family}`"
          size="small"
          :type="state.modelFamily === family ? 'primary' : 'default'"
          @click="state.modelFamily = family"
        >
          {{ family }}
        </el-button>
      </div>

      <el-table
        v-if="state.modelDisplay === 'table'"
        :data="filteredModelRows"
        :border="false"
        style="width: 100%"
        size="small"
      >
        <el-table-column prop="category" label="Category" width="180" />
        <el-table-column prop="family" label="Models Family" width="140" />
        <el-table-column prop="modelName" label="Model" min-width="240" />
        <el-table-column align="center" label="Available On This PC" width="170">
          <template #default="scope">
            <el-tag v-if="scope.row.available" type="success" size="small">Installed</el-tag>
            <span v-else class="text-xs text-gray-500">No</span>
          </template>
        </el-table-column>
        <el-table-column prop="quant" label="Best Quant" width="110" />
        <el-table-column prop="context" label="Context" width="100" />
        <el-table-column prop="expectedTokSec" label="Expected tok/s" width="130" />
        <el-table-column prop="firstToken" label="First Token" width="120" />
        <el-table-column label="Suggested Features" min-width="220">
          <template #default="scope">
            <div class="flex flex-wrap gap-1.5">
              <el-tag
                v-for="feat in scope.row.features"
                :key="`${scope.row.category}-${feat}`"
                size="small"
                type="info"
                effect="plain"
                >{{ feat }}</el-tag
              >
            </div>
          </template>
        </el-table-column>
      </el-table>

      <div v-else class="grid grid-cols-1 md:grid-cols-2 gap-3">
        <el-card
          v-for="row in filteredCategoryRows"
          :key="`card-${row.category}`"
          shadow="never"
          class="category-card"
        >
          <template #header>
            <div class="flex justify-between items-center">
              <span class="font-medium">{{ row.category }}</span>
              <el-tag size="small" effect="plain">{{ row.context }}</el-tag>
            </div>
          </template>

          <div class="mb-2 text-xs">
            <span class="text-gray-500">Expected tok/s:</span>
            <span class="font-medium"> {{ row.expectedTokSec }}</span>
          </div>
          <div class="mb-2 text-xs">
            <span class="text-gray-500">First token:</span>
            <span class="font-medium"> {{ row.firstToken }}</span>
          </div>
          <div class="mb-2 text-xs">
            <span class="text-gray-500">Quant:</span>
            <span class="font-medium"> {{ row.quant }}</span>
          </div>

          <div class="mb-2">
            <div class="text-xs text-gray-500 mb-1">Suggested Models</div>
            <div class="flex flex-wrap gap-1.5">
              <el-tag v-for="name in row.suggested" :key="`s-${row.category}-${name}`" size="small">
                {{ name }}
              </el-tag>
            </div>
          </div>

          <div class="mb-2">
            <div class="text-xs text-gray-500 mb-1">Available On This PC</div>
            <div class="flex flex-wrap gap-1.5">
              <template v-if="row.available.length">
                <el-tag
                  v-for="name in row.available"
                  :key="`a-${row.category}-${name}`"
                  size="small"
                  type="success"
                >
                  {{ name }}
                </el-tag>
              </template>
              <template v-else>
                <span class="text-xs text-gray-500">No local match</span>
              </template>
            </div>
          </div>

          <div>
            <div class="text-xs text-gray-500 mb-1">Suggested Features</div>
            <div class="flex flex-wrap gap-1.5">
              <el-tag
                v-for="feat in row.features"
                :key="`f-${row.category}-${feat}`"
                size="small"
                effect="plain"
                type="info"
              >
                {{ feat }}
              </el-tag>
            </div>
          </div>
        </el-card>
      </div>
    </el-card>
  </div>
</template>

<script lang="ts" setup>
  import { computed, onMounted, reactive, watch } from 'vue'
  import IPC from '@/util/IPC'
  import { BrewStore } from '@/store/brew'
  import { AppStore } from '@/store/app'
  import { AppModuleSetup } from '@/core/Module'

  type LocalModelItem = {
    name: string
    size?: string
  }

  type CatalogItem = {
    name: string
    size?: string
  }

  const state = reactive<{
    fetching: boolean
    error: string
    localModels: LocalModelItem[]
    catalog: Record<string, CatalogItem[]>
    pcReport: any
    gpuTab: string
    modelCategory: string
    modelDisplay: 'table' | 'cards'
    modelFamily: string
  }>({
    fetching: false,
    error: '',
    localModels: [],
    catalog: {},
    pcReport: {},
    gpuTab: '0',
    modelCategory: 'all',
    modelDisplay: 'table',
    modelFamily: 'all'
  })

  const { tab } = AppModuleSetup('ollama')

  const callOllama = async (method: string, ...args: any[]) => {
    return new Promise<any>((resolve, reject) => {
      try {
        IPC.send('app-fork:ollama', method, ...args).then((key: string, res: any) => {
          IPC.off(key)
          resolve(res)
        })
      } catch (e) {
        reject(e)
      }
    })
  }

  const toArr = (v: any) => {
    if (!v) return []
    return Array.isArray(v) ? v : [v]
  }

  const fmtGB = (bytesLike: any) => {
    const n = Number(bytesLike || 0)
    if (!n) return 'Unknown'
    return `${Math.round((n / 1024 / 1024 / 1024) * 100) / 100} GB`
  }

  const fmtFromMiB = (miB: any) => {
    const n = Number(miB || 0)
    if (!n) return 'Unknown'
    return `${Math.round((n / 1024) * 100) / 100} GB`
  }

  const fmtGHzFromMHz = (mhz: any) => {
    const n = Number(mhz || 0)
    if (!n) return 'Unknown'
    return `${Math.round((n / 1000) * 100) / 100} GHz`
  }

  const ddrTypeName = (smbiosType: any, memoryType: any) => {
    const map: Record<number, string> = {
      20: 'DDR',
      21: 'DDR2',
      24: 'DDR3',
      26: 'DDR4',
      34: 'DDR5'
    }
    const sm = Number(smbiosType || 0)
    const mt = Number(memoryType || 0)
    return map[sm] || map[mt] || 'DDR (Unknown Gen)'
  }

  const cleanVendor = (name: string) => {
    const n = (name || '').toLowerCase()
    if (n.includes('nvidia')) return 'NVIDIA'
    if (n.includes('amd') || n.includes('advanced micro devices')) return 'AMD'
    if (n.includes('intel')) return 'Intel'
    return name || 'Unknown'
  }

  const extractModel = (fullName: string, vendor: string) => {
    if (!fullName) return 'Unknown'
    const re = new RegExp(`^${vendor}\\s*`, 'i')
    return fullName.replace(re, '').trim() || fullName
  }

  const inferGpuMemoryType = (raw: any) => {
    const text = `${raw?.Name || ''} ${raw?.VideoProcessor || ''}`
    const t = text.toUpperCase()

    const m = text.match(/(GDDR\dX?|DDR\d|HBM\d?)/i)
    if (m?.[1]) {
      return m[1].toUpperCase()
    }

    // Best-effort inference by common GPU naming families when direct data is missing.
    if (t.includes('QUADRO M')) return 'GDDR5'
    if (t.includes('GTX 9')) return 'GDDR5'
    if (t.includes('GTX 10')) return 'GDDR5'
    if (t.includes('RTX 20')) return 'GDDR6'
    if (t.includes('RTX 30')) return 'GDDR6/GDDR6X'
    if (t.includes('RTX 40')) return 'GDDR6/GDDR6X'
    if (t.includes('RTX A')) return 'GDDR6'

    if (t.includes('RX 4') || t.includes('RX 5')) return 'GDDR5'
    if (t.includes('RX 6') || t.includes('RX 7') || t.includes('RADEON PRO W')) return 'GDDR6'

    if (t.includes('ARC A')) return 'GDDR6'

    return 'DDR?'
  }

  const osDisplay = computed(() => {
    const os = state.pcReport?.os || {}
    const caption = `${os?.Caption || 'Windows'}`
    const build = `${os?.BuildNumber || ''}`
    const arch = `${window.Server?.Arch || os?.OSArchitecture || 'Unknown'}`
    let winVer = ''
    const b = Number(build || 0)
    if (caption.toLowerCase().includes('windows')) {
      if (caption.includes('11') || b >= 22000) winVer = '11'
      else if (caption.includes('10') || b >= 10240) winVer = '10'
      else if (caption.includes('8.1')) winVer = '8.1'
      else if (caption.includes('8')) winVer = '8'
      else if (caption.includes('7')) winVer = '7'
    }
    const buildPart = build ? ` (Build ${build})` : ''
    if (winVer) {
      return `Windows ${winVer}${buildPart} ${arch}`
    }
    return `${caption}${buildPart} ${arch}`
  })

  const cpuInfoText = computed(() => {
    const cpu = toArr(state.pcReport?.cpu)
    if (!cpu.length) {
      const fallback = navigator.hardwareConcurrency || '-'
      return {
        model: 'Unknown',
        processorsCount: 1,
        formula: `1 * ${fallback} cores = ${fallback}`,
        threads: `${fallback} * 1 = ${fallback}`
        ,
        logicalProcessors: `${fallback}`,
        speed: 'Unknown'
      }
    }
    const sockets = cpu.length
    const totalCores = cpu.reduce((s: number, c: any) => s + Number(c?.NumberOfCores || 0), 0)
    const totalThreads = cpu.reduce(
      (s: number, c: any) => s + Number(c?.NumberOfLogicalProcessors || 0),
      0
    )
    const totalMhz = cpu.reduce((s: number, c: any) => s + Number(c?.MaxClockSpeed || 0), 0)
    const coresPerSocket = Math.round((totalCores / sockets) * 100) / 100
    const threadsPerCore = totalCores ? Math.round((totalThreads / totalCores) * 100) / 100 : 0
    return {
      model: `${cpu?.[0]?.Name || 'Unknown'}`,
      processorsCount: sockets,
      formula: `${sockets} * ${coresPerSocket} cores = ${totalCores}`,
      threads: `${totalCores} * ${threadsPerCore} = ${totalThreads}`,
      logicalProcessors: `${totalThreads}`,
      speed: fmtGHzFromMHz(sockets ? totalMhz / sockets : 0)
    }
  })

  const ramInfoText = computed(() => {
    const mods = toArr(state.pcReport?.memory)
    const arrays = toArr(state.pcReport?.memoryArray)
    const totalSlots = Number(arrays?.[0]?.MemoryDevices || 0) || mods.length
    const installed = mods.length
    const total = mods.reduce((s: number, m: any) => s + Number(m?.Capacity || 0), 0)
    const speed = Number(mods?.[0]?.ConfiguredClockSpeed || mods?.[0]?.Speed || 0)
    const ddr = ddrTypeName(mods?.[0]?.SMBIOSMemoryType, mods?.[0]?.MemoryType)
    if (!installed) {
      return `Unknown`
    }
    const sameSize = mods.every((m: any) => Number(m?.Capacity || 0) === Number(mods[0]?.Capacity || 0))
    const sizeExpr = sameSize
      ? `${installed} * ${fmtGB(mods[0]?.Capacity)}`
      : mods.map((m: any) => fmtGB(m?.Capacity)).join(' + ')
    const speedText = speed ? ` @ ${speed} MHz` : ''
    return `${totalSlots} slots (${installed} installed): ${sizeExpr} = ${fmtGB(total)} ${ddr}${speedText}`
  })

  const gpuRows = computed(() => {
    const gpus = toArr(state.pcReport?.gpu)
    const nvidia = toArr(state.pcReport?.nvidia)
    return gpus
      .map((g: any) => {
        const vendor = cleanVendor(`${g?.AdapterCompatibility || ''}`)
        const model = extractModel(`${g?.Name || ''}`, vendor)
        const h = Number(g?.CurrentHorizontalResolution || 0)
        const v = Number(g?.CurrentVerticalResolution || 0)
        let vram = fmtGB(g?.AdapterRAM)
        let ramType = inferGpuMemoryType(g)

        if (vendor === 'NVIDIA' && nvidia.length > 0) {
          const nrow = nvidia.find((n: any) => {
            const a = `${n?.Name || ''}`.toLowerCase()
            const b = `${g?.Name || ''}`.toLowerCase()
            return a.includes(b) || b.includes(a)
          })
          if (nrow?.MemoryTotalMiB) {
            vram = fmtFromMiB(nrow.MemoryTotalMiB)
          }
        }

        return {
          cardName: `${vendor} ${model}`.trim(),
          vendor,
          model,
          vram: `${vram} ${ramType}`.trim(),
          driverVersion: `${g?.DriverVersion || 'Unknown'}`,
          resolution: h && v ? `${h}x${v}` : 'Unknown',
          processor: `${g?.VideoProcessor || 'Unknown'}`
        }
      })
      .filter((g: any) => g.vendor !== 'Unknown' || g.model !== 'Unknown')
  })

  const appStore = AppStore()
  const brewStore = BrewStore()

  const currentService = computed(() => {
    const current = appStore.config.server?.ollama?.current
    return brewStore
      .module('ollama')
      .installed.find((o) => o.path === current?.path && o.version === current?.version)
  })

  const ollamaServiceRunning = computed(() => {
    return !!currentService.value?.run
  })

  const goToOllamaServiceTab = () => {
    tab.value = 0
  }

  const localModelsCount = computed(() => state.localModels.length)

  const totalSystemRamGB = computed(() => {
    const mods = toArr(state.pcReport?.memory)
    const total = mods.reduce((s: number, m: any) => s + Number(m?.Capacity || 0), 0)
    if (!total) return 0
    return Math.round((total / 1024 / 1024 / 1024) * 100) / 100
  })

  const maxGpuVramGB = computed(() => {
    let max = 0
    const nvidia = toArr(state.pcReport?.nvidia)
    nvidia.forEach((row: any) => {
      const gb = Number(row?.MemoryTotalMiB || 0) / 1024
      if (gb > max) max = gb
    })
    if (max > 0) return Math.round(max * 100) / 100

    const gpus = toArr(state.pcReport?.gpu)
    gpus.forEach((g: any) => {
      const gb = Number(g?.AdapterRAM || 0) / 1024 / 1024 / 1024
      if (gb > max) max = gb
    })
    return Math.round(max * 100) / 100
  })

  const performanceTier = computed(() => {
    const ram = totalSystemRamGB.value
    const vram = maxGpuVramGB.value
    const logical = Number(cpuInfoText.value.logicalProcessors || 0)

    if (vram >= 16 || (ram >= 64 && logical >= 24)) {
      return {
        key: 'high',
        label: 'High',
        maxVramText: vram ? `${vram} GB` : 'Unknown',
        systemRamText: ram ? `${ram} GB` : 'Unknown',
        tok: '35 - 90',
        firstToken: '0.25 - 0.8s',
        quant: 'Q6_K / Q8_0',
        context: '16K - 32K'
      }
    }
    if (vram >= 8 || (ram >= 32 && logical >= 12)) {
      return {
        key: 'mid',
        label: 'Balanced',
        maxVramText: vram ? `${vram} GB` : 'Unknown',
        systemRamText: ram ? `${ram} GB` : 'Unknown',
        tok: '18 - 45',
        firstToken: '0.5 - 1.4s',
        quant: 'Q4_K_M / Q5_K_M',
        context: '8K - 16K'
      }
    }
    return {
      key: 'low',
      label: 'Entry',
      maxVramText: vram ? `${vram} GB` : 'Unknown',
      systemRamText: ram ? `${ram} GB` : 'Unknown',
      tok: '6 - 20',
      firstToken: '1.2 - 3.5s',
      quant: 'Q4_K_M / Q3_K_M',
      context: '4K - 8K'
    }
  })

  const machineHint = computed(() => {
    const ramText = ramInfoText.value
    const hasDDR5 = ramText.toLowerCase().includes('ddr5')
    const hasDDR4 = ramText.toLowerCase().includes('ddr4')
    const lowRam = ramText.includes('= 8 GB') || ramText.includes('= 4 GB')
    if (lowRam) {
      return 'Device profile: low memory capacity. Suggested: start with 3B-7B models, then scale up after latency checks.'
    }
    if (hasDDR5) {
      return 'Device profile: high-speed memory (DDR5). Suggested: start with 14B and move higher based on latency.'
    }
    if (hasDDR4) {
      return 'Device profile: balanced. Suggested: 7B-14B models for good quality/performance trade-off.'
    }
    return 'Device profile: lightweight. Suggested: 3B-7B models for better speed and lower memory pressure.'
  })

  const summaryItems = computed(() => [
    { key: 'OS', value: osDisplay.value },
    { key: 'PC Provider', value: `${state.pcReport?.computer?.Manufacturer || 'Unknown'}` },
    { key: 'PC Model', value: `${state.pcReport?.computer?.Model || 'Unknown'}` },
    { key: 'Processor', value: cpuInfoText.value.model },
    { key: 'Processors Count', value: `${cpuInfoText.value.processorsCount}` },
    { key: 'CPU Speed', value: cpuInfoText.value.speed },
    { key: 'CPU', value: cpuInfoText.value.formula },
    { key: 'Logical Processors', value: cpuInfoText.value.logicalProcessors },
    { key: 'Threads', value: cpuInfoText.value.threads },
    { key: 'PC RAM', value: ramInfoText.value },
    { key: 'Local Models', value: `${localModelsCount.value}` }
  ])

  const allCatalogNames = computed(() => {
    const names: string[] = []
    Object.values(state.catalog).forEach((arr) => {
      arr.forEach((item) => {
        if (item?.name) names.push(item.name)
      })
    })
    return Array.from(new Set(names))
  })

  const normalize = (name: string) => name.toLowerCase()

  const pickByKeywords = (keywords: string[]) => {
    const result = allCatalogNames.value.filter((name) => {
      const n = normalize(name)
      return keywords.some((k) => n.includes(k))
    })
    return result.slice(0, 5)
  }

  const hasLocalMatch = (target: string) => {
    const t = normalize(target)
    return state.localModels.some((model) => {
      const n = normalize(model.name)
      return n === t || n.startsWith(`${t}:`) || t.startsWith(`${n}:`)
    })
  }

  const modelFamily = (name: string) => {
    let base = `${name || ''}`.toLowerCase().trim()
    if (base.includes('/')) {
      const arr = base.split('/')
      base = arr[arr.length - 1]
    }
    base = base.split(':')[0]
    const family = base.split(/[-_]/)[0]
    return family || base || 'unknown'
  }

  const categoryRows = computed(() => {
    const tier = performanceTier.value
    const rows = [
      {
        category: 'Coding',
        suggested: pickByKeywords(['coder', 'code', 'codellama', 'deepseek-coder', 'starcoder']),
        features: ['Fill-in-middle', 'Long context', 'Tool call'],
        context: tier.context,
        quant: tier.quant,
        expectedTokSec: tier.tok,
        firstToken: tier.firstToken
      },
      {
        category: 'Documents/RAG',
        suggested: pickByKeywords(['embed', 'bge', 'nomic-embed', 'mxbai', 'e5', 'minilm']),
        features: ['Embeddings', 'Rerank', 'Citation style'],
        context: tier.key === 'low' ? '4K - 8K' : '8K - 32K',
        quant: tier.key === 'high' ? 'Q8_0' : 'Q4_K_M',
        expectedTokSec: tier.key === 'high' ? '120 - 600 (embed/s)' : '50 - 250 (embed/s)',
        firstToken: 'N/A'
      },
      {
        category: 'General Chat',
        suggested: pickByKeywords(['llama', 'qwen', 'gemma', 'mistral', 'phi']),
        features: ['Reasoning', 'JSON mode', 'Multilingual'],
        context: tier.context,
        quant: tier.quant,
        expectedTokSec: tier.tok,
        firstToken: tier.firstToken
      },
      {
        category: 'Vision',
        suggested: pickByKeywords(['vision', 'llava', 'moondream', 'bakllava', 'vl']),
        features: ['Image input', 'OCR', 'Chart QA'],
        context: tier.key === 'low' ? '4K - 8K' : '8K - 16K',
        quant: tier.key === 'high' ? 'Q5_K_M / Q6_K' : 'Q4_K_M',
        expectedTokSec: tier.key === 'high' ? '12 - 35' : '5 - 18',
        firstToken: tier.key === 'high' ? '0.6 - 1.8s' : '1.5 - 4.0s'
      }
    ]
    return rows.map((row) => {
      const fallback = row.suggested.length ? row.suggested : ['No catalog match']
      return {
        category: row.category,
        suggested: fallback,
        available: row.suggested.filter((name) => hasLocalMatch(name)),
        quant: row.quant,
        context: row.context,
        expectedTokSec: row.expectedTokSec,
        firstToken: row.firstToken,
        features: row.features
      }
    })
  })

  const categoryOptions = computed(() => {
    const base = [{ label: 'All Categories', value: 'all' }]
    const items = categoryRows.value.map((row) => ({
      label: row.category,
      value: row.category
    }))
    return [...base, ...items]
  })

  const filteredCategoryRows = computed(() => {
    if (state.modelCategory === 'all') {
      return categoryRows.value
    }
    return categoryRows.value.filter((row) => row.category === state.modelCategory)
  })

  const categoryFilteredModelRows = computed(() => {
    const rows: any[] = []
    filteredCategoryRows.value.forEach((row) => {
      row.suggested
        .filter((name: string) => name && name !== 'No catalog match')
        .forEach((name: string) => {
          rows.push({
            category: row.category,
            family: modelFamily(name),
            modelName: name,
            available: hasLocalMatch(name),
            quant: row.quant,
            context: row.context,
            expectedTokSec: row.expectedTokSec,
            firstToken: row.firstToken,
            features: row.features
          })
        })
    })
    return rows
  })

  const familyOptions = computed(() => {
    const all = categoryFilteredModelRows.value.map((r) => r.family)
    return Array.from(new Set(all))
  })

  const filteredModelRows = computed(() => {
    if (state.modelFamily === 'all') {
      return categoryFilteredModelRows.value
    }
    return categoryFilteredModelRows.value.filter((row) => row.family === state.modelFamily)
  })

  watch(
    () => state.modelCategory,
    () => {
      state.modelFamily = 'all'
    }
  )

  const fetchLocalModels = async () => {
    const brewStore = BrewStore()
    const appStore = AppStore()
    await brewStore.module('ollama').fetchInstalled()

    const current = appStore.config.server?.ollama?.current
    const service = brewStore
      .module('ollama')
      .installed.find((o) => o.path === current?.path && o.version === current?.version)

    if (!service) {
      state.localModels = []
      return
    }
    const res = await callOllama('allModel', JSON.parse(JSON.stringify(service)))
    state.localModels = res?.data ?? []
  }

  const fetchCatalog = async () => {
    const res = await callOllama('fetchAllModels')
    state.catalog = res?.data ?? {}
  }

  const fetchPcReport = async () => {
    const res = await callOllama('pcReport')
    state.pcReport = res?.data ?? {}
  }

  const refresh = async () => {
    state.fetching = true
    state.error = ''
    try {
      await Promise.all([fetchLocalModels(), fetchCatalog(), fetchPcReport()])
    } catch (e: any) {
      state.error = e?.message || 'Failed to load checker data.'
    } finally {
      state.fetching = false
    }
  }

  onMounted(() => {
    refresh().then().catch()
  })
</script>

<style scoped>
  .summary-row {
    display: flex;
    gap: 6px;
    align-items: center;
  }
  .summary-key {
    color: #6b7280;
    min-width: 120px;
  }
  .summary-value {
    word-break: break-all;
  }
</style>

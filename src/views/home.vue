<template>
  <div class="min-h-screen flex items-center flex-col gap-y-3 p-4 overflow-auto">
    <!-- <h1 class="text-2xl md:text-5xl font-bold mb-5 text-foreground">Travel Wardrobe</h1> -->
    <HyperText
      text="Travel Wardrobe"
      class="text-4xl font-bold"
      :duration="800"
      :animate-on-load="true"
    />
    <Stepper class="flex w-full items-start gap-2" v-model="step">
      <StepperItem
        v-for="item in steps"
        :key="item.step"
        v-slot="{ state }"
        class="relative flex w-full flex-col items-center justify-center"
        :step="item.step"
      >
        <StepperSeparator
          v-if="item.step !== steps[steps.length - 1].step"
          class="absolute left-[calc(50%+20px)] right-[calc(-50%+10px)] top-5 block h-0.5 shrink-0 rounded-full bg-muted group-data-[state=completed]:bg-primary"
        />

        <StepperTrigger as-child>
          <Button
            :variant="state === 'completed' || state === 'active' ? 'default' : 'outline'"
            size="icon"
            class="z-10 !rounded-full shrink-0"
            :class="[
              state === 'active' && 'ring-2 ring-ring ring-offset-2 ring-offset-background ',
            ]"
            :disabled="state !== 'completed' && item.step !== step"
          >
            <Check v-if="state === 'completed'" class="size-5" />
            <Circle v-if="state === 'active'" />
            <Dot v-if="state === 'inactive'" />
          </Button>
        </StepperTrigger>

        <div class="mt-1 flex flex-col items-center text-center">
          <StepperTitle
            :class="[state === 'active' && 'text-primary']"
            class="text-sm font-semibold transition lg:text-base"
          >
            {{ item.title }}
          </StepperTitle>
        </div>
      </StepperItem>
    </Stepper>

    <div class="grid grid-cols-1 gap-4 w-full sm:px-12 mt-4 flex-1">
      <!-- User Input Section -->
      <UserInputCard
        :is-active="step === 1"
        @generate="handleUserInputGenerate"
        @next-tab="handleNextTab"
      />

      <!-- Information Section -->
      <InformationCard
        :is-active="step === 2"
        @generate="handleInformationGenerate"
        :data="informationData"
        class="overflow-y-auto"
      />

      <!-- Wardrobe Section -->
      <WardrobeCard :is-active="step === 3" class="overflow-y-auto" :data="resultData" />
    </div>
  </div>
  <!-- 添加全屏loading组件 -->
  <FullScreenLoading :visible="isLoading" :text="loadingText" />
</template>

<script setup>
import { ref } from 'vue'
import UserInputCard from '@/components/UserInputCard.vue'
import InformationCard from '@/components/InformationCard.vue'
import WardrobeCard from '@/components/WardrobeCard.vue'
import { Button } from '@/components/ui/button'
import { toast } from 'vue-sonner'
import HyperText from '@/components/ui/hyper-text/HyperText.vue'
import FullScreenLoading from '@/components/ui/loading/FullScreenLoading.vue'
import {
  Stepper,
  StepperItem,
  StepperSeparator,
  StepperTitle,
  StepperTrigger,
} from '@/components/ui/stepper'
import { Check, Circle, Dot } from 'lucide-vue-next'

// 添加全局loading状态
const isLoading = ref(false)
const loadingText = ref('Generating Results')

const steps = [
  {
    step: 1,
    title: 'User Input',
  },
  {
    step: 2,
    title: 'Information',
  },
  {
    step: 3,
    title: 'Results',
  },
]
const step = ref(1)
const userInputData = ref({})
const informationData = ref({})
const resultData = ref()
const getImageTypeFromUrl = (url) => {
  if (url.includes('.')) {
    return url.split('.').pop()
  }
  return 'jpg'
}
const handleNextTab = () => {
  step.value = step.value + 1
}

const handleUserInputGenerate = async (data) => {
  isLoading.value = true
  loadingText.value = 'Processing Your Request'

  const response = await fetch(`${import.meta.env.VITE_API_URL}/admin/submit`, {
    method: 'POST',
    headers: {
      'ngrok-skip-browser-warning': 'true',
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(data),
  }).finally(() => {
    isLoading.value = false
  })
  const result = await response.json()
  const { errNo, errMsg } = result
  if (!response.ok || errNo !== 0) {
    const text = errMsg || response.statusText || 'Request failed'
    toast.error(text)
    return
  }
  userInputData.value = data
  informationData.value = result.data
  if (informationData.value?.result3) {
    informationData.value.result3 = informationData.value.result3.map((item) => {
      const { base64, ...rest } = item
      const urlKey = Object.keys(rest).find((key) => key.startsWith('text'))
      const type = urlKey ? getImageTypeFromUrl(urlKey) : 'jpg'
      const base64String = `data:image/${type};base64,${base64}`
      return {
        base64: base64String,
        url: item[urlKey],
      }
    })
  }
  handleNextTab()
}

const handleInformationGenerate = async () => {
  isLoading.value = true
  loadingText.value = 'Creating Your Wardrobe'
  const params = new URLSearchParams({
    id: informationData.value?.id,
  })
  const response = await fetch(
    `${import.meta.env.VITE_API_URL}/admin/getResults?${params.toString()}`,
    {
      headers: {
        'ngrok-skip-browser-warning': 'true',
      },
    },
  ).finally(() => {
    isLoading.value = false
  })
  const result = await response.json()
  const { errNo, errMsg } = result
  if (!response.ok || errNo !== 0) {
    const text = errMsg || response.statusText || 'Request failed'
    toast.error(text)
    return
  }
  resultData.value = (result.data?.list || []).map((item) => {
    return {
      ...item,
      type: getImageTypeFromUrl(item.image || item.url),
    }
  })
  handleNextTab()
}
</script>

<style></style>

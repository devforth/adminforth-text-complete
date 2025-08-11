<template>
  <div class="flex bg-gray-50 border border-gray-300 rounded-lg focus:ring-blue-500 
      focus:border-blue-500 w-full p-2.5 dark:bg-gray-700 dark:border-gray-600 
      dark:focus:ring-blue-500 dark:focus:border-blue-500 relative max-w-full">
    <SuggestionInput 
      ref="suggestionInputRef"
      class="w-full !border-none text-gray-900 text-sm dark:placeholder-gray-400 dark:text-white whitespace-normal mr-28"
      v-model="currentValue"
      :type="column.type"
      :completionRequest="complete"
      :debounceTime="meta.debounceTime"
      @completion-approved="handleCompletionApproved"
    />
    <div class="absolute right-2 bottom-1">
      <Tooltip v-if="isUntouched || (!currentValue.trim() && !isFocused)">
        <button
          @click.stop="handleFocus"
          @mousedown.prevent
          class="text-white bg-gradient-to-r from-purple-500 via-purple-600 to-purple-700 hover:bg-gradient-to-br focus:ring-4 focus:outline-none focus:ring-purple-300 dark:focus:ring-purple-800 
            font-medium rounded-lg text-sm w-6 h-6 flex items-center justify-center">
            <IconMagic class="w-4 h-4"/>
        </button>
        <template #tooltip>
          {{ $t('Start completion') }}
        </template>
      </Tooltip>  

      <Spinner v-else-if="isLoading" class="w-6 h-6" lightFill="#000000" darkFill="#ffffff" />
      <Tooltip v-else-if="isFocused">
        <button
          @click.stop="approveCompletion"
          @mousedown.prevent
          :class="[
            'text-white bg-gradient-to-r from-purple-500 via-purple-600 to-purple-700 hover:bg-gradient-to-br focus:ring-4 focus:outline-none focus:ring-purple-300 dark:focus:ring-purple-800',
            'font-medium rounded-lg text-xs flex items-center justify-center py-1 px-1  ',
            buttonText === approveCompletionValue ? 'w-16' : 'w-18'
          ]">
            <div
            class="flex items-center justify-center"
            v-if="buttonText === approveCompletionValue"
            >
              <IconArrowRightThin class="mt-0.5 w-5 h-5 text-white"/>
              <span class="ml-1 px-1 h-4 flex items-center justify-center rounded border bg-white text-black text-[10px] font-mono shadow-inner shadow-sm border-gray-300 dark:bg-gray-700 dark:text-white dark:border-gray-500">
                <p class="mt-0.5">Tab</p>
              </span>
            </div>
            <div
            class="flex items-center justify-center"
            v-else-if="buttonText === approveNextWordValue"
            >
              <IconArrowRightThin class="mt-0.5 w-5 h-5 text-white" />
              <span class="ml-1 px-1 h-4 flex items-center justify-center rounded border bg-white text-black text-[10px] font-mono shadow-inner shadow-sm border-gray-300 dark:bg-gray-700 dark:text-white dark:border-gray-500">
                <p class="mt-0.5">Ctrl</p>
              </span> 
              <span class="ml-1 text-white">+</span>
              <span class="ml-1 px-1 h-4 flex items-center justify-center rounded border bg-white text-black text-[10px] font-mono shadow-inner shadow-sm border-gray-300 dark:bg-gray-700 dark:text-white dark:border-gray-500">
                  <IconArrowRightThin class="w-3.5 h-3.5" />
              </span>
            </div>
        </button>
        <template #tooltip>
          {{ $t(tooltipText) }}
        </template>
      </Tooltip>
     
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, watch, Ref, computed } from 'vue';
import { callAdminForthApi } from '@/utils';
import { AdminForthColumnCommon } from '@/types/Common';
import { Spinner, Tooltip } from '@/afcl';
import { IconMagic, IconCheck, IconArrowRightThin } from '@iconify-prerendered/vue-mdi';
import SuggestionInput from 'vue-suggestion-input';
import 'vue-suggestion-input/dist/style.css';

const props = defineProps<{
  column: AdminForthColumnCommon,
  record: any,
  meta: any,
}>();

const emit = defineEmits([
  'update:value',
]);
const approveCompletionValue:string='TAB';
const approveNextWordValue:string='CTRL + ->'
const isLoading = ref<boolean>(false);
const isUntouched = ref<boolean>(true);
const isFocused = ref<boolean>(false);
const currentValue: Ref<string> = ref('');
const suggestionInputRef = ref<InstanceType<typeof SuggestionInput> | null>(null);
const buttonText = ref<string>(approveCompletionValue);

const tooltipText = computed(() => 
  buttonText.value === approveCompletionValue ? 'Approve completion' : 'Approve next word'
);

function handleCompletionApproved(type: 'all' | 'word') {
  if(buttonText.value === approveCompletionValue && type === 'all') {
    buttonText.value = approveNextWordValue;
  } else if (buttonText.value === approveNextWordValue && type === 'word') {
    buttonText.value = approveCompletionValue;
  }
}

onMounted(() => {
  const value = props.record[props.column.name] || '';
  currentValue.value = value;
  if (value.trim()) {
    isUntouched.value = false;
  }
  if (suggestionInputRef.value) {
    const editor = suggestionInputRef.value.$el.querySelector('.ql-editor');
    if (editor) {
      editor.addEventListener('focus', handleFocus);
      editor.addEventListener('blur', handleBlur);
    }
  }
});

watch(() => currentValue.value, (value) => {
  emit('update:value', value);
});

watch(() => props.record, (value) => {
  const val = value[props.column.name] || '';
  currentValue.value = val;
  if (val.trim()) {
    isUntouched.value = false;
  }
});

async function complete(textBeforeCursor: string) {
  isLoading.value = true;
  isUntouched.value = false;
  const res = await callAdminForthApi({
      path: `/plugin/${props.meta.pluginInstanceId}/doComplete`,
      method: 'POST',
      body: {
        record: {...props.record, [props.column.name]: textBeforeCursor},
      },
  });

  isLoading.value = false;
  
  return res.completion;
}

const approveCompletion = async () => {
  if (suggestionInputRef.value) {
    if( buttonText.value === approveCompletionValue) {
      await suggestionInputRef.value.approveCompletion('all');
    } else {
      await suggestionInputRef.value.approveCompletion('word');
    }
  }
  buttonText.value === approveCompletionValue ? buttonText.value = approveNextWordValue : buttonText.value = approveCompletionValue;
}

function handleFocus() {
  isFocused.value = true;
  if (suggestionInputRef.value) {
    const editor = suggestionInputRef.value.$el.querySelector('.ql-editor');
    if (editor) {
      editor.focus();
    }
  }
}

function handleBlur() {
  isFocused.value = false;
}

</script>

<style>
.ql-editor::before {
  @apply text-gray-500 dark:text-gray-400 text-sm font-normal font-sans !important;
}
</style>

<template>
  <div class="flex bg-gray-50 border border-gray-300 rounded-lg focus:ring-blue-500 
      focus:border-blue-500 w-full p-2.5 dark:bg-gray-700 dark:border-gray-600 
      dark:focus:ring-blue-500 dark:focus:border-blue-500 relative max-w-full">
    <SuggestionInput 
      ref="suggestionInputRef"
      class="w-full !border-none text-gray-900 text-sm dark:placeholder-gray-400 dark:text-white whitespace-normal mr-14"
      v-model="currentValue"
      :type="column.type"
      :completionRequest="complete"
      :debounceTime="meta.debounceTime"
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
          class="text-white bg-gradient-to-r from-purple-500 via-purple-600 to-purple-700 hover:bg-gradient-to-br focus:ring-4 focus:outline-none focus:ring-purple-300 dark:focus:ring-purple-800
            font-medium rounded-lg text-xs w-14 h-6 flex items-center justify-center">
            <IconArrowRightThin class="w-5 h-5"/>
            <span class="scale-75 border border-white rounded-sm px-0.5 bg-white/25">TAB</span>
        </button>
        <template #tooltip>
          {{ $t('Approve completion') }}
        </template>
      </Tooltip>
     
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, watch, Ref  } from 'vue';
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

const isLoading = ref<boolean>(false);
const isUntouched = ref<boolean>(true);
const isFocused = ref<boolean>(false);
const currentValue: Ref<string> = ref('');
const suggestionInputRef = ref<InstanceType<typeof SuggestionInput> | null>(null);

onMounted(() => {
  currentValue.value = props.record[props.column.name] || '';
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
  currentValue.value = value[props.column.name] || '';
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
    console.log('✋ approveCompletion', suggestionInputRef.value);
    await suggestionInputRef.value.approveCompletion('all');
  }
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

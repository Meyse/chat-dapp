<script lang="ts">
// Component: src/lib/components/CustomDropdown.svelte
// Description: A custom-styled dropdown component mimicking WhatsApp's UI style.

  import { createEventDispatcher, tick } from 'svelte';
  import { fly } from 'svelte/transition';

  interface Option {
    id: string | number | null;
    name: string;
    enabled: boolean;
  }

  export let options: Option[] = [];
  export let selectedId: string | number | null = null;
  export let label: string = '';
  export let placeholder: string = '-- Select --';
  export let disabled: boolean = false;

  let isOpen = false;
  let selectedOptionName: string = placeholder;
  let dropdownElement: HTMLElement;

  const dispatch = createEventDispatcher<{ change: string | number | null }>();

  // Update selected name when selectedId prop changes externally
  $: {
      const foundOption = options.find(opt => opt.id === selectedId);
      selectedOptionName = foundOption ? foundOption.name : placeholder;
  }

  function toggleDropdown() {
    if (disabled) return;
    isOpen = !isOpen;
  }

  async function selectOption(option: Option) {
    if (!option.enabled || disabled) return;
    selectedId = option.id;
    selectedOptionName = option.name;
    isOpen = false;
    await tick(); // Ensure state updates before dispatching
    dispatch('change', selectedId);
  }

  function handleClickOutside(event: MouseEvent) {
    if (dropdownElement && !dropdownElement.contains(event.target as Node)) {
      isOpen = false;
    }
  }
  
  // Function to handle keydown events on list items
  function handleKeydown(event: KeyboardEvent, option: Option) {
      if (event.key === 'Enter') {
          selectOption(option);
      }
  }
</script>

<svelte:window on:click={handleClickOutside} />

<div class="relative w-full" bind:this={dropdownElement}>
  {#if label}
    <label class="block text-sm font-medium text-gray-700 mb-1">{label}</label>
  {/if}
  <button 
    type="button" 
    class="relative w-full bg-white border border-gray-300 rounded-md shadow-sm pl-3 pr-10 py-2 text-left cursor-default focus:outline-none focus:ring-1 focus:ring-green-500 focus:border-green-500 sm:text-sm disabled:bg-gray-100 disabled:cursor-not-allowed"
    class:cursor-not-allowed={disabled}
    class:bg-gray-100={disabled}
    on:click={toggleDropdown}
    aria-haspopup="listbox"
    aria-expanded={isOpen}
    {disabled}
  >
    <span class="block truncate" class:text-gray-500={selectedId === null}>{selectedOptionName}</span>
    <span class="absolute inset-y-0 right-0 flex items-center pr-2 pointer-events-none">
      <!-- Heroicon name: solid/selector -->
      <svg class="h-5 w-5 text-gray-400" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" aria-hidden="true">
        <path fill-rule="evenodd" d="M10 3a1 1 0 01.707.293l3 3a1 1 0 01-1.414 1.414L10 5.414 7.707 7.707a1 1 0 01-1.414-1.414l3-3A1 1 0 0110 3zm-3.707 9.293a1 1 0 011.414 0L10 14.586l2.293-2.293a1 1 0 011.414 1.414l-3 3a1 1 0 01-1.414 0l-3-3a1 1 0 010-1.414z" clip-rule="evenodd" />
      </svg>
    </span>
  </button>

  {#if isOpen}
    <ul 
      class="absolute z-10 mt-1 w-full bg-white shadow-lg max-h-60 rounded-md py-1 text-base ring-1 ring-black ring-opacity-5 overflow-auto focus:outline-none sm:text-sm"
      tabindex="-1"
      role="listbox"
      aria-labelledby="listbox-label"
      transition:fly={{ y: -5, duration: 150 }}
    >
      {#each options as option (option.id)}
        <li 
          class="text-gray-900 cursor-default select-none relative py-2 pl-3 pr-9 group"
          class:cursor-not-allowed={!option.enabled}
          class:text-gray-400={!option.enabled}
          role="option"
          aria-selected={option.id === selectedId}
          on:click={() => selectOption(option)}
          on:keydown={(e) => handleKeydown(e, option)}
          tabindex="0"
        >
          <span class="block truncate" class:font-semibold={option.id === selectedId}>{option.name}</span>
          {#if !option.enabled}
            <span class="absolute inset-y-0 right-0 flex items-center pr-4 text-xs text-gray-400">Coming Soon</span>
          {:else if option.id === selectedId}
            <span class="absolute inset-y-0 right-0 flex items-center pr-4 text-green-600">
              <!-- Heroicon name: solid/check -->
              <svg class="h-5 w-5" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" aria-hidden="true">
                <path fill-rule="evenodd" d="M16.707 5.293a1 1 0 010 1.414l-8 8a1 1 0 01-1.414 0l-4-4a1 1 0 011.414-1.414L8 12.586l7.293-7.293a1 1 0 011.414 0z" clip-rule="evenodd" />
              </svg>
            </span>
          {/if}
        </li>
      {/each}
    </ul>
  {/if}
</div>
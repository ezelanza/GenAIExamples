<!--
  Copyright (c) 2024 Intel Corporation

  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at

     http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.
-->

<script lang="ts">
  import { Button, Helper, Input, Label, Modal } from "flowbite-svelte";
  import { ExclamationCircleOutline } from "flowbite-svelte-icons";
  import DropFile from "./dropFile.svelte";
  import SpinLoading from "./assets/spinLoading.svelte";
  import { kb_id, loading } from "./shared/Store.js";
  import { Textarea } from "flowbite-svelte";
  import { createEventDispatcher } from "svelte";
  import { getNotificationsContext } from "svelte-notifications";

  const { addNotification } = getNotificationsContext();
  const dispatch = createEventDispatcher();

  let activePanel = 1;
  let message = "";
  let formModal = false;
  let currentIdx = 1;

  function togglePanel(index: number) {
    if (activePanel === index) {
      return;
    } else {
      currentIdx = index;
      if (
        (currentIdx === 2 && message !== "") ||
        (currentIdx === 1 && $kb_id !== "")
      ) {
        formModal = true;
      } else {
        activePanel = currentIdx;
      }
    }
  }

  function panelExchange() {
    if (currentIdx === 1) {
      kb_id.set("");
      dispatch("clearMsg", { status: true });
    } else if (currentIdx === 2) {
      message = "";
      dispatch("clearMsg", { status: true });
    }
    activePanel = currentIdx;
    formModal = false;
  }

  function generateFaq() {
    if ($kb_id === "" && message === "") {
      addNotification({
        text: "Please upload content first",
        position: "bottom-center",
        type: "warning",
        removeAfter: 3000,
      });
      return;
    } else {
      loading.set(true);
      if ($kb_id !== "") {
        dispatch("generateFaq", { mode: "file", value: $kb_id });
      } else if (message !== "") {
        dispatch("generateFaq", { mode: "text", value: message });
      }
    }
  }
</script>

<div class="w-full h-full relative">
  {#each [1, 2] as panelIndex}
    <div id="accordion-open" data-accordion="open">
      <h2 id={`accordion-open-heading-${panelIndex}`}>
        <div class="flex flex-row border-b border-gray-200">
          <div class="bg-[#0054ae] w-9"></div>
          <button
            type="button"
            class="flex items-center bg-[#00285a] justify-between w-full p-5 font-medium text-white hover:bg-opacity-90 gap-3"
            on:click={() => togglePanel(panelIndex)}
            aria-expanded={panelIndex === activePanel}
            aria-controls={`accordion-open-body-${panelIndex}`}
          >
            <span class="flex items-center"
              >{panelIndex === 1 ? "Paste Text" : "Upload File" }</span
            >
            <svg
              data-accordion-icon
              class="w-3 h-3 rotate-180 shrink-0"
              aria-hidden="true"
              xmlns="http://www.w3.org/2000/svg"
              fill="none"
              viewBox="0 0 10 6"
            >
              <path
                stroke="currentColor"
                stroke-linecap="round"
                stroke-linejoin="round"
                stroke-width="2"
                d="M9 5 5 1 1 5"
              />
            </svg>
          </button>
        </div>
      </h2>
      <div
        id={`accordion-open-body-${panelIndex}`}
        class:open={panelIndex === activePanel}
        class="px-2 transition-all duration-300 ease-in-out overflow-hidden"
        aria-labelledby={`accordion-open-heading-${panelIndex}`}
      >
        {#if panelIndex === activePanel}
          {#if panelIndex === 1}
            <Textarea
              data-testid="sum-input"
              id="textarea-id"
              class="xl:h-[30rem] xl:my-4"
              placeholder="Copy the text information you need to generate FAQs."
              rows="13"
              name="message"
              bind:value={message}
            />
          {:else}
            <DropFile />
          {/if}
        {/if}
      </div>
    </div>
  {/each}
  {#if $loading}
    <button
      type="submit"
      class="xl:my-12 gap-3 inline-flex items-center px-5 py-2.5 text-sm font-medium text-center text-white bg-gray-300 mt-2"
    >
      <SpinLoading />
      Generating
    </button>
  {:else}
    <button
      type="submit"
      data-testid="sum-click"
      class="xl:my-12 inline-flex items-center px-5 py-2.5 text-sm font-medium text-center text-white bg-blue-700 mt-2 focus:ring-4 focus:ring-blue-200 dark:focus:ring-blue-900 hover:bg-blue-800"
       on:click={() => generateFaq()}
    >
      Generate FAQs
    </button>
  {/if}
</div>

<Modal bind:open={formModal} size="xs" autoclose={false} class="w-full">
  <ExclamationCircleOutline
    class="mx-auto mb-4 text-gray-400 w-12 h-12 dark:text-gray-200"
  />
  {#if currentIdx === 1}
    <h3 class="mb-5 text-lg font-normal text-gray-500 dark:text-gray-400">
      The currently uploaded file will be cleared.
    </h3>
  {:else if currentIdx === 2}
    <h3 class="mb-5 text-lg font-normal text-gray-500 dark:text-gray-400">
      The current content will be cleared.
    </h3>
  {/if}

  <Button type="submit" class="w-full" on:click={() => panelExchange()}
    >Confirm</Button
  >
</Modal>

<style>
  .open {
    height: auto;
    padding-top: 10px;
    padding-bottom: 10px;
  }
</style>

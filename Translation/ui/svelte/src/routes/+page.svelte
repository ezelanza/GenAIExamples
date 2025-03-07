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
  import Header from "$lib/header.svelte";
  import { fetchLanguageResponse } from "$lib/shared/Network.js";
  import type { Language } from "./types.js";
  import { languagesList } from "$lib/shared/constant.js";
  import LoadingAnimation from "$lib/assets/loadingAnimation.svelte";

  // Set default language
  let langFrom: string = "zh";
  let langTo: string = "en";
  let languages: Language[] = languagesList;
  // Initialize disabled state of input
  let inputDisabled: boolean = false;
  // Initialize input and output
  let input: string = "";
  let output: string = "";
  let loading = false;

  function decodeEscapedBytes(str: string): string {
    const byteArray = str
      .split("\\x")
      .slice(1)
      .map((byte) => parseInt(byte, 16));
    return new TextDecoder("utf-8").decode(new Uint8Array(byteArray));
  }

  function decodeUnicode(str: string): string {
    return str.replace(/\\u[\dA-Fa-f]{4}/g, (match) => {
      return String.fromCharCode(parseInt(match.replace(/\\u/g, ""), 16));
    });
  }

  const handelTranslate = async () => {
    loading = true;
    output = "";
    const eventSource = await fetchLanguageResponse(input, langFrom, langTo);

    eventSource.addEventListener("message", (e: any) => {
      let res = e.data;
      if (res === "[DONE]") {
        loading = false;
      } else {
        let Msg = JSON.parse(res).choices[0].text;
        if (/\\x[\dA-Fa-f]{2}/.test(Msg)) {
          Msg = decodeEscapedBytes(Msg);
        } else if (/\\u[\dA-Fa-f]{4}/.test(Msg)) {
          Msg = decodeUnicode(Msg);
        }

        if (Msg !== "</s>") {
          output += Msg.replace(/\\n/g, "\n");
        }
      }
    });
    eventSource.stream();
  };

  let timer;

  $: if ((input || langFrom || langTo) && input !== "") {
    if (langFrom === langTo) {
      output = input;
    } else {
      clearTimeout(timer);
      timer = setTimeout(handelTranslate, 1000);
    }
  }
</script>

<div>
  <Header />
  <div class="mt-10 flex flex-col items-center">
    <div class="w-[70%]">
      <div class="flex items-center mx-2">
        <select
          class="p-4 m-2 shadow-md rounded w-full border-[#0000001f] hover:border-[#0054ae]"
          name="lang-from"
          id="lang-from"
          bind:value={langFrom}
        >
          {#each languages as language}
            <option value={language.shortcode}
              >{#if language.flagUnicode}{language.flagUnicode} |
              {/if}{language.name}</option
            >
          {/each}
        </select>

        <select
          class="p-4 m-2 shadow-md rounded w-full border-[#0000001f] hover:border-[#0054ae]"
          name="lang-to"
          id="lang-to"
          bind:value={langTo}
        >
          {#each languages as language}
            <option value={language.shortcode}
              >{#if language.flagUnicode}{language.flagUnicode} |
              {/if}{language.name}</option
            >
          {/each}
        </select>
      </div>
      <div class="translate-textareas">
        <textarea
          class="grow"
          disabled={inputDisabled}
          name="input"
          id="translateinput"
          rows="25"
          placeholder="Input"
          bind:value={input}
          data-testid="translate-input"
        />
        <textarea
          readonly
          name="output"
          class="bg-[#f5f5f5] grow disabled"
          id="translateoutput"
          rows="25"
          placeholder="Translation"
          bind:value={output}
          data-testid="translate-output"
        />
      </div>
    </div>
    {#if loading}
      <LoadingAnimation />
    {/if}
  </div>
</div>

<style>
  .translate-textareas {
    padding: 8px;
    display: flex;
    flex-direction: row;
    align-items: center;
    width: 100%;
  }

  textarea {
    resize: none;
    margin: 8px;
    padding: 8px;

    font-size: 16px;

    border-radius: 12px;
    border: solid rgba(128, 0, 128, 0) 4px;
    box-shadow: 0 0 8px rgba(0, 0, 0, 0.19);

    transition: 0.1s linear;
  }

  #translateinput:hover {
    border: solid #0054ae 4px;
  }
</style>

<script lang="ts">
  import {
    createDevice,
    TransportEvent,
    TempoEvent,
    TimeNow,
    type Device,
    type Parameter,
  } from "@rnbo/js";
  import "./app.css";
  import type { IPatcher, IParameterDescription } from "@rnbo/js";
  import patcher from "./../public/max/breakerR.export.json";
  import Pad from "./lib/Pad.svelte";
  import { text } from "svelte/internal";

  let WAContext = window.AudioContext || window.webkitAudioContext;
  let context = new WAContext();
  let isSetup = false;

  // Unlock audio on iOS
  if (context.state === "suspended" && "ontouchstart" in window) {
    // Unlock it
    context.resume();
    let unlock = () => {
      context.resume().then(function () {
        document.body.removeEventListener("touchstart", unlock);
        document.body.removeEventListener("touchend", unlock);
      });
    };

    document.body.addEventListener("touchstart", unlock, false);
    document.body.addEventListener("touchend", unlock, false);
  }

  let device: Device;
  let outputNode: GainNode;
  let params: Parameter[] = [];
  let tempo: number = 60;
  let gain: number = 50;

  const setupAudio = async () => {
    const devPatch: IPatcher = patcher;
    device = await createDevice({ context: context, patcher: devPatch });

    let presets = patcher.presets || [];
    if (presets.length < 1) {
      console.log("No presets defined");
    }

    device.setPreset(presets[0].preset);
    params = device.parameters;

    // Create gain node and connect it to audio output
    outputNode = context.createGain();
    outputNode.connect(context.destination);

    // This connects the device to audio output, but you may still need to call context.resume()
    // from a user-initiated function.
    device.node.connect(outputNode);
    isSetup = true;
  };

  let promise = setupAudio();

  const startAudio = () => {
    context.resume();
    let tempoEvent = new TempoEvent(TimeNow, tempo);
    device.scheduleEvent(tempoEvent);
    let transportEvent = new TransportEvent(TimeNow, 1);
    device.scheduleEvent(transportEvent);
  };

  function changeParam(index: number) {
    if (params[index].value === 2) {
      params[index].value = 0;
    } else {
      params[index].value = params[index].value + 1;
    }
  }

  function setTempo(hTempo: number) {
    let tempoEvent = new TempoEvent(TimeNow, hTempo);
    device.scheduleEvent(tempoEvent);
  }

  function pamToDisp(input: number) {
    if (input === 1) return "X";
    else if (input === 2) return "O";
    else return "B";
  }

  $: {
    if (isSetup) setTempo(tempo);
  }
  $: {
    if (isSetup) outputNode.gain.value = (gain / 100) * 2;
  }

  let textColor = ["text-slate-200", "text-red-500", "text-blue-500"];
</script>

<main class="font-sans text-center">
  <h1 class=" font-bold text-7xl mt-8">X O</h1>
  <p class="text-xl opacity-50">A simple drum machine</p>
  {#await promise}
    <p>loading...</p>
  {:then}
    <div class="square grid grid-cols-4 max-w-lg p-2 mx-auto gap-4 mt-4">
      {#each params as pam, index}
        <button
          class="square font-bold text-4xl lg:text-7xl text-center p-4 hover:opacity-50 bg-slate-200"
          on:click={() => changeParam(index)}
        >
          <span class={textColor[pam.value]}>{pamToDisp(pam.value)}</span
          ></button
        >
      {/each}
    </div>

    <button class="text-white bg-black p-2 mt-2" on:click={startAudio}
      >Start Audio</button
    >
    <div>
      <input type="range" name="tempo" max="240" min="10" bind:value={tempo} />
      <label for="tempo">Tempo {tempo}</label>
    </div>
    <div>
      <input type="range" name="gainVal" max="100" min="0" bind:value={gain} />
      <label for="gainVal">Gain: {gain}</label>
    </div>
  {:catch error}
    <p>error: {error.message}</p>
  {/await}
</main>

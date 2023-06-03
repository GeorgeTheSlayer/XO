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
  let params: Parameter[];
  let tempo: number = 60;
  let gain: number = 50;
  let counter = 0;
  let isRunning = false;
  let xClass = "text-black";
  let oClass = "text-black";

  const setupAudio = async () => {
    let rawPatcher = await fetch("/max/breakerR.export.json");
    let devPatch: IPatcher = await rawPatcher.json();
    device = await createDevice({ context: context, patcher: devPatch });

    let presets = devPatch.presets || [];
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

    device.messageEvent.subscribe((ev) => {
      if (ev.tag === "counter") {
        counter = ev.payload;
      }
    });
  };

  let promise = setupAudio();
  promise.then(() => {
    console.log("Audio setup complete");
    isSetup = true;
  });

  const startAudio = () => {
    if (context.state === "suspended") {
      context.resume();
      let tempoEvent = new TempoEvent(TimeNow, tempo);
      device.scheduleEvent(tempoEvent);
      let transportEvent = new TransportEvent(TimeNow, 1);
      device.scheduleEvent(transportEvent);
      isRunning = true;
    } else {
      context.suspend();
      let transportEvent = new TransportEvent(TimeNow, 0);
      device.scheduleEvent(transportEvent);
      isRunning = false;
    }
  };

  function changeParam(index: number) {
    console.log("index: ", index);
    if (isSetup) {
      if (params[index].value == 2) {
        params[index].value = 0;
      } else {
        params[index].value = params[index].value + 1;
      }
      console.log(params[index].value);
    }
  }

  function setTempo(hTempo: number) {
    let tempoEvent = new TempoEvent(TimeNow, hTempo);
    device.scheduleEvent(tempoEvent);
  }

  function pamToDisp(input: number) {
    if (input === 1) return "X";
    else if (input === 2) return "O";
    else return " ";
  }

  function counterToDisp(input: number, count: number = counter) {
    if (input == params[count].value) {
      return true;
    } else {
      return false;
    }
  }

  $: {
    if (isSetup) setTempo(tempo);
  }
  $: {
    if (isSetup) outputNode.gain.value = (gain / 100) * 2;
  }

  $: {
    if (isSetup) {
      if (counterToDisp(1, counter)) {
        xClass = "text-red-500";
      } else {
        xClass = "text-black";
      }
      if (counterToDisp(2, counter)) {
        oClass = "text-blue-500";
      } else {
        oClass = "text-black";
      }
    }
  }
</script>

<main class="font-sans text-center">
  <body>
    {#await promise}
      <p>Setting Up Audio</p>
    {:then}
      <h1 class="font-bold text-7xl mt-8">
        <span class={xClass}>X</span>
        <span class={oClass}>0</span>
      </h1>
      <p class="text-xl opacity-50">A simple drum machine</p>
      <div class="square grid grid-cols-4 max-w-lg p-2 mx-auto gap-4 mt-4">
        {#each params as pam, index}
          <button
            class="square {counter == index
              ? 'bg-black text-white'
              : 'bg-slate-200 text-black'} font-bold text-4xl lg:text-7xl text-center p-4 hover:opacity-50"
            on:click={() => changeParam(index)}
          >
            {pamToDisp(pam.value)}
          </button>
        {/each}
      </div>
      <div class="flex mx-auto w-full max-w-lg justify-evenly">
        <div class="flex flex-col">
          <input
            type="range"
            name="tempo"
            max="240"
            min="10"
            bind:value={tempo}
          />
          <label for="tempo">Tempo {tempo}</label>
        </div>
        <div class="flex flex-col">
          <input
            type="range"
            name="gainVal"
            max="100"
            min="0"
            bind:value={gain}
          />
          <label for="gainVal">Gain: {gain}</label>
        </div>
      </div>
      {#if !isRunning}
        <button
          class="text-white bg-black p-4 mt-2 max-w-lg"
          on:click={startAudio}
        >
          <svg
            xmlns="http://www.w3.org/2000/svg"
            class="icon icon-tabler icon-tabler-player-play-filled"
            width="24"
            height="24"
            viewBox="0 0 24 24"
            stroke-width="2"
            stroke="currentColor"
            fill="none"
            stroke-linecap="round"
            stroke-linejoin="round"
          >
            <path stroke="none" d="M0 0h24v24H0z" fill="none" />
            <path
              d="M6 4v16a1 1 0 0 0 1.524 .852l13 -8a1 1 0 0 0 0 -1.704l-13 -8a1 1 0 0 0 -1.524 .852z"
              stroke-width="0"
              fill="currentColor"
            />
          </svg>
        </button>
      {:else}
        <button
          class="text-white bg-black p-4 mt-2 max-w-lg"
          on:click={startAudio}
        >
          <svg
            xmlns="http://www.w3.org/2000/svg"
            class="icon icon-tabler icon-tabler-player-pause-filled"
            width="24"
            height="24"
            viewBox="0 0 24 24"
            stroke-width="2"
            stroke="currentColor"
            fill="none"
            stroke-linecap="round"
            stroke-linejoin="round"
          >
            <path stroke="none" d="M0 0h24v24H0z" fill="none" />
            <path
              d="M9 4h-2a2 2 0 0 0 -2 2v12a2 2 0 0 0 2 2h2a2 2 0 0 0 2 -2v-12a2 2 0 0 0 -2 -2z"
              stroke-width="0"
              fill="currentColor"
            />
            <path
              d="M17 4h-2a2 2 0 0 0 -2 2v12a2 2 0 0 0 2 2h2a2 2 0 0 0 2 -2v-12a2 2 0 0 0 -2 -2z"
              stroke-width="0"
              fill="currentColor"
            />
          </svg>
        </button>
      {/if}
    {/await}
  </body>
</main>

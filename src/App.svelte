<script lang="ts">

  //wtf
  import {
    createDevice,
    TransportEvent,
    TempoEvent,
    TimeNow,
    type ExternalDataInfo,
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
  let keyVal = false;
  let xClass = "text-black";
  let oClass = "text-black";

  const setupAudio = async () => {
    let rawPatcher = await fetch("/max/breakerR.export.json");
    let devPatch: IPatcher = await rawPatcher.json();
    device = await createDevice({ context: context, patcher: devPatch });

    let dependencies = await fetch("/max/dependencies.json");
    dependencies = await dependencies.json();
    const results = await device.loadDataBufferDependencies(dependencies);
    results.forEach((result) => {
      if (result.type === "success") {
        console.log(`Successfully loaded buffer with id ${result.id}`);
      } else {
        console.log(
          `Failed to load buffer with id ${result.id}, ${result.error}`
        );
      }
    });

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
  promise.then(async () => {
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
    if (isSetup) {
      if (params[index].value == 2) {
        params[index].value = 0;
      } else {
        params[index].value = params[index].value + 1;
      }
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

  function resetParams() {
    params.forEach((param) => {
      param.value = 0;
    });
    keyVal = !keyVal;
  }

  function randomizeParams() {
    params.forEach((param) => {
      param.value = Math.floor(Math.random() * 3);
    });
    keyVal = !keyVal;
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


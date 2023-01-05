<script>
  import { getContext, onDestroy } from "svelte";
  import Button from "../node_modules/@budibase/bbui/src/Button/Button.svelte";

  const STAGES = {
    IDLE: ["IDLE", "VoiceOver"],
    RECORDING: ["RECORDING", "Circle"],
    DONE: ["DONE", "DeleteOutline"],
  };

  const EXTENSIONS = {
    "audio/wav": "wav",
    "audio/ogg": "ogg",
  };

  let fieldApi;
  let fieldState;
  let sound_url;
  let stage = STAGES.IDLE;
  let icon = "VoiceOver";
  let download;
  let recorder;
  let recordedChunks = [];

  export let type = "audio/wav";
  export let filename = "recording";
  export let size;
  export let disabled;
  export let field;

  const { styleable, API } = getContext("sdk");
  const component = getContext("component");
  const formContext = getContext("form");
  const formStepContext = getContext("form-step");
  const formApi = formContext?.formApi;

  $: formStep = formStepContext ? $formStepContext || 1 : 1;

  $: formField = formApi?.registerField(
    field,
    "attachment",
    null,
    false,
    null,
    formStep
  );
  
  $: unsubscribe = formField?.subscribe((value) => {
    fieldState = value?.fieldState;
    fieldApi = value?.fieldApi;
    console.log(value)
    value?.fieldState?.value != null && value?.fieldState?.value.length>0 ? (sound_url = value?.fieldState?.value[0].url, stage=STAGES.DONE,icon=STAGES.DONE[1] ) : null;
  });


  onDestroy(() => {
    fieldApi?.deregister();
    unsubscribe?.();
  });


  async function handleRecord() {
    switch (stage) {
      case STAGES.IDLE:
        icon = STAGES.IDLE[1];
        const stream = await navigator.mediaDevices.getUserMedia({
          audio: true,
        });

        recorder = new MediaRecorder(stream);
        icon = STAGES.RECORDING[1];

        recorder.addEventListener("dataavailable", function (evt) {
          if (evt.data.size > 0) {
            recordedChunks.push(evt.data);
          }
        });

        recorder.addEventListener("stop", function () {
          download = {
            link: URL.createObjectURL(new Blob(recordedChunks)),
            filename,
          };
        });
        recorder.start();
        stage = STAGES.RECORDING;
        icon = STAGES.RECORDING[1];
        break;
      case STAGES.RECORDING:
        recorder.stop();
        handleClick();
        stage = STAGES.DONE;
        icon = STAGES.DONE[1];
        break;
      case STAGES.DONE:
        deleteAttachments([fieldState?.value[0].key]);
        icon = STAGES.IDLE[1];
        stage = STAGES.IDLE;
        break;
    }
  }

  function updateFieldValue(response) {
    fieldApi?.setValue(response);
  }

  async function deleteAttachments(keys) {
    try {
      return await API.deleteAttachments({
        keys: keys,
        tableId: formContext?.dataSource?.tableId,
      }).then(function (response) {
        fieldApi?.setValue(null);
      });
    } catch (error) {
      return [];
    }
  }

  async function handleClick() {
    let data = new FormData();
    let newArray = [];
    newArray.push(
      new Blob(recordedChunks, {
        type: type,
      })
    );
    await new Promise(r => setTimeout(r, 500));
    data.append(
      "file",
      new File(recordedChunks, filename + "." + EXTENSIONS[type], { type: type })
    );
    try {
      return await API.uploadAttachment({
        data,
        tableId: formContext?.dataSource?.tableId,
      }).then(function (response) {
        updateFieldValue(response);
      });
    } catch (error) {
      console.log(error);
      return [];
    }
  }
</script>
<svelte:head>
	<!-- elements go here -->
<meta http-equiv="Content-Security-Policy" content="media-src 'self' data: blob: *;">
</svelte:head>
<div class="container" use:styleable={$component.styles}>

 
  {#if !formContext}
    <div class="placeholder">Form components need to be wrapped in a form</div>
  {:else}
    <Button {size} {disabled} {icon} cta on:click={handleRecord}>
      {#if stage == STAGES.IDLE}
        Record
      {:else if stage == STAGES.RECORDING}
        Stop
      {:else if stage == STAGES.DONE}
        Delete
      {/if}
    </Button>
    {#if stage === STAGES.RECORDING}
      <div class="record-icon recording" />
    {/if}

    {#if stage === STAGES.DONE}
      <audio
        controls
        class="audio-player"
        {type}
        title={`${filename}.${EXTENSIONS[type]}`}
        src={download?.link ?? sound_url}
      />
    {/if}
  {/if}
</div>

<style>
  .container {
    display: flex;
    align-items: center;
  }

  .record-icon {
    margin-left: 10px;
    height: 12px;
    width: 12px;
    border-radius: 100%;
    background: rgb(211, 21, 16);
  }

  .record-icon.recording {
    animation: pulse 2s infinite;
  }

  @keyframes pulse {
    0% {
      transform: scale(0.95);
      box-shadow: 0 0 0 0 rgba(211, 21, 16, 0.7);
    }

    70% {
      transform: scale(1);
      box-shadow: 0 0 0 10px rgba(211, 21, 16, 0);
    }

    100% {
      transform: scale(0.95);
      box-shadow: 0 0 0 0 rgba(211, 21, 16, 0);
    }
  }
</style>

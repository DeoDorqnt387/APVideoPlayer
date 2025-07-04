<script>
  import { Icon } from 'svelte-icons-pack';
  import { FaSolidPlay, FaSolidPause, FaSolidForward, FaSolidBackward } from 'svelte-icons-pack/fa';
  import { VscMute, VscUnmute, VscSettingsGear } from 'svelte-icons-pack/vsc';
  import { RiMediaFullscreenLine, RiMediaFullscreenExitFill } from 'svelte-icons-pack/ri';

  let videoElement = null;
  let videoContainer = null;
  let paused = true;
  let muted = false;
  let volume = 1;
  let isFullscreen = false;

  let currentTime = 0;
  let duration = 0;
  let isDragging = false; 
  let showVolume = false;
  let showSettings = false;

  let showControls = true;
  let controlsTimeout;

  function showControlsTemporarily() {
    if (paused) {
      showControls = true;
      clearTimeout(controlsTimeout);
      return;
    }
    showControls = true;
    clearTimeout(controlsTimeout);
    controlsTimeout = setTimeout(() => {
      showControls = false;
    }, 2000);
  }

  function togglePlay() {
    paused = !paused;
    if (paused) {
      showControls = true;
      clearTimeout(controlsTimeout);
    } else {
      showControls = false;
    }
  }

  /*
    function toggleMute(){
      muted = !muted;
    }
  */


  function skipForward() {
    if (videoElement) {
      videoElement.currentTime += 10;
    }
  }

  function skipBackward() {
    if (videoElement) {
      videoElement.currentTime -= 10;
    }
  }

  function formatVideo(timeInSeconds) {
    const hour = Math.floor(timeInSeconds / 3600);
    const minutes = Math.floor((timeInSeconds % 3600) / 60);
    const seconds = Math.floor(timeInSeconds % 60);
    const hourStr = hour > 0 ? hour.toString().padStart(2, "0") + ":" : "";
    const minStr = minutes.toString().padStart(2, "0");
    const secStr = seconds.toString().padStart(2, "0");
    return `${hourStr}${minStr}:${secStr}`;
  }

  function handleProgressClick(event) {
    if (!videoElement || !duration) return;
    const progressBar = event.currentTarget;
    const rect = progressBar.getBoundingClientRect();
    const clickX = event.clientX - rect.left;
    const progressBarWidth = rect.width;
    const clickedTime = (clickX / progressBarWidth) * duration;
    videoElement.currentTime = clickedTime;
  }

  function handleProgressMouseDown(event) {
    isDragging = true;
    handleProgressClick(event);
  }

  function handleProgressMouseMove(event) {
    if (!isDragging || !videoElement || !duration) return;
    const progressBar = event.currentTarget;
    const rect = progressBar.getBoundingClientRect();
    const clickX = event.clientX - rect.left;
    const progressBarWidth = rect.width;
    const clickedTime = Math.max(0, Math.min((clickX / progressBarWidth) * duration, duration));
    videoElement.currentTime = clickedTime;
  }

  function handleProgressMouseUp() {
      isDragging = false;
  }

  function handleVolumeChange(event) {
    if (videoElement) {
      const input = event.target;
      volume = parseFloat(input.value);
      videoElement.volume = volume;
      muted = volume === 0;
    }
  }

  function handleGlobalMouseUp() {
      isDragging = false;
  }

  function handleVolumeIconClick(event) {
    showVolume = !showVolume;
    event.stopPropagation(); // Prevent window click from immediately closing
  }

  function handleSettingsIconClick(event) {
    showSettings = !showSettings;
    event.stopPropagation();
  }

  function handleFullscreen() {
    if (videoContainer) {
      if (!document.fullscreenElement) {
        videoContainer.requestFullscreen();
      } else {
        document.exitFullscreen();
      }
    }
  }

  function onFullscreenChange() {
    isFullscreen = !!document.fullscreenElement;
  }

  // Keyboard shortcuts handler
  function handleKeyPress(event) {
    if (document.activeElement && ['INPUT', 'TEXTAREA', 'SELECT'].includes(document.activeElement.tagName)) return;
    switch (event.key) {
      case ' ': event.preventDefault(); togglePlay(); break;
      case 'f': handleFullscreen(); break;
      case 'm': muted = !muted; break;
      case 'ArrowUp':
        event.preventDefault();
        volume = Math.min(1, volume + 0.1);
        if (videoElement) videoElement.volume = volume;
        muted = volume === 0;
        break;
      case 'ArrowDown':
        event.preventDefault();
        volume = Math.max(0, volume - 0.1);
        if (videoElement) videoElement.volume = volume;
        muted = volume === 0;
        break;
      case 'ArrowLeft':
        event.preventDefault();
        if (videoElement) videoElement.currentTime = Math.max(0, videoElement.currentTime - 5);
        break;
      case 'ArrowRight':
        event.preventDefault();
        if (videoElement) videoElement.currentTime = Math.min(duration, videoElement.currentTime + 5);
        break;
    }
  }

  let error = null;
  function handleVideoError(e) {
    // @ts-ignore
    let code = e?.target?.error?.code;
    let errMsg = 'Unknown error';
    switch (code) {
      case 1: errMsg = 'ERR_ABORT: Video loading aborted.'; break;
      case 2: errMsg = 'ERR_NETWORK: Network error.'; break;
      case 3: errMsg = 'ERR_DECODE: Video decode error.'; break;
      case 4: errMsg = 'ERR_SRC_NOT_SUPPORTED: Video format not supported.'; break;
      default: errMsg = 'ERR_UNKNOWN: Unknown error.';
    }
    error = errMsg;
  }

  import { onMount, onDestroy } from 'svelte';
  onMount(() => {
    window.addEventListener('keydown', handleKeyPress);
    return () => window.removeEventListener('keydown', handleKeyPress);
  });

</script>

<svelte:window
  on:mouseup={handleGlobalMouseUp}
  on:click={() => { showVolume = false; }}
  on:fullscreenchange={onFullscreenChange}
/>

<div
  bind:this={videoContainer}
  class="fixed inset-0 z-50 bg-gradient-to-br from-[#18122B] via-[#393053] to-[#443C68] flex items-center justify-center group{showControls ? '' : ' hide-cursor'}"
  on:mousemove={showControlsTemporarily}
  on:mouseleave={() => { showControls = false; }}
  role="region"
>
    <video 
      bind:this={videoElement}
      bind:paused
      bind:muted
      bind:currentTime
      bind:duration
      bind:volume
      class="w-full h-full max-h-full max-w-full object-contain rounded-xl shadow-2x1 bg-black"
      style="aspect-ratio: 16/9;"
      on:click={togglePlay}
      on:error={handleVideoError}
      playsinline
    >
    <track kind="captions" />
    <source src="/zerokarahaji.mkv" type="video/mp4">
  </video>

    <!-- Controls -->
    <div class="bg-gradient-to-t from-black/80 via-black/60 to-transparent absolute bottom-0 left-0 w-full p-3 rounded-b-xl shadow-lg {showControls ? 'animate-fade-in' : 'animate-fade-out'}" style="pointer-events: {showControls ? 'auto' : 'none'};">
      {#if error}
        <div class="absolute top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 bg-[#18122B]/95 border border-red-500 text-red-300 rounded-lg px-6 py-4 shadow-xl z-50 text-lg font-mono animate-fade-in">
          {error}
        </div>
      {/if}
      {#if showControls || controlsTimeout}
        <!-- Top controls: Centered Rewind, Play/Pause, Forward -->
        <div class="flex items-center justify-center gap-4 mb-4">
          <button on:click={skipBackward} class="hover:bg-[#443C68]/80 p-2 rounded-full transition duration-200">
              <Icon color="#fef9ec" src={FaSolidBackward} size="28px" />
          </button>
          <button 
              class="hover:bg-[#605985]/80 text-white p-4 rounded-full border-2 border-[#fef9ec] transition duration-200 shadow-lg" 
              style="background-color: #605985" 
              on:click={togglePlay}
          >
              <Icon color="#fef9ec" src={paused ? FaSolidPlay : FaSolidPause} size="32px" />
          </button>
          <button on:click={skipForward} class="hover:bg-[#443C68]/80 p-2 rounded-full transition duration-200">
              <Icon color="#fef9ec" src={FaSolidForward} size="28px" />
          </button>
        </div>

        <!-- Bottom controls: Progress bar left, duration center, right controls (settings, volume, fullscreen) -->
        <div class="flex items-center justify-between gap-4">
          <!-- Progress bar -->
          <div class="flex-1 flex items-center">
            <div 
                class="w-full h-2 bg-gradient-to-r from-[#605985]/60 via-[#fef9ec]/40 to-[#18122B]/60 rounded-full cursor-pointer relative group shadow-inner"
                on:click={handleProgressClick}
                on:mousedown={handleProgressMouseDown}
                on:mousemove={handleProgressMouseMove}
                on:mouseup={handleProgressMouseUp}
                role="slider"
                tabindex="0"
                aria-valuenow={currentTime}
                aria-valuemin={0}
                aria-valuemax={duration}
                aria-label="Video progress bar"
                on:keydown={(e) => {
                  if (!videoElement || !duration) return;
                  if (e.key === 'ArrowLeft') videoElement.currentTime = Math.max(0, videoElement.currentTime - 5);
                  if (e.key === 'ArrowRight') videoElement.currentTime = Math.min(duration, videoElement.currentTime + 5);
                }}
            >
                <!-- Progress Fill -->
                <div 
                    class="h-full rounded-full transition-all duration-100 shadow-lg"
                    style="background: #605985; width: {duration > 0 ? (currentTime / duration) * 100 : 0}%"
                ></div>
                <!-- Progress Handle (Thumb) -->
                <div 
                    class="absolute top-1/2 transform -translate-y-1/2 w-4 h-4 rounded-full opacity-0 group-hover:opacity-100 transition-opacity duration-200 border-2 border-[#fef9ec] shadow"
                    style="background-color: #605985; left: calc({duration > 0 ? (currentTime / duration) * 100 : 0}% - 8px)"
                ></div>
            </div>
          </div>
          <!-- Duration center -->
          <span class="text-white text-sm px-2 py-0.5 font-mono tracking-wider drop-shadow whitespace-nowrap">{formatVideo(currentTime)} / {formatVideo(duration)}</span>
          <!-- Right controls -->
          <div class="flex items-center gap-2 ml-2">
            <button on:click={handleSettingsIconClick} class="flex items-center gap-2 hover:bg-[#443C68]/80 p-2 rounded-full transition duration-200">
                <Icon color="#fef9ec" src={VscSettingsGear} size="22px" />
            </button>
            {#if showSettings}
            <div class="absolute bottom-14 right-32 z-30 bg-[#18122B]/95 border border-[#605985] rounded-lg shadow-lg p-4 min-w-[180px] flex flex-col gap-2 animate-fade-in">
                <span class="text-[#fef9ec] font-semibold text-base mb-2">Settings</span>
                <div class="flex items-center justify-between">
                  <span class="text-[#fef9ec] text-sm">Quality</span>
                  <select class="bg-[#393053] text-[#fef9ec] rounded px-2 py-1 text-sm border border-[#605985]">
                    <option>Auto</option>
                    <option>1080p</option>
                    <option>720p</option>
                    <option>480p</option>
                  </select>
                </div>
                <div class="flex items-center justify-between">
                  <span class="text-[#fef9ec] text-sm">Subtitles</span>
                  <select class="bg-[#393053] text-[#fef9ec] rounded px-2 py-1 text-sm border border-[#605985]">
                    <option>Off</option>
                    <option>Turkish</option>
                    <option>English</option>
                  </select>
                </div>
            </div>
            {/if}
            <div class="relative flex flex-col items-center justify-end">
                <button on:click={handleVolumeIconClick} class="volume-control hover:bg-[#443C68]/80 p-2 rounded-full transition duration-200">
                    <Icon color="#fef9ec" src={muted ? VscMute : VscUnmute} size="22px" />
                </button>
                {#if showVolume}
                <div class="absolute bottom-full right-1/2 translate-x-1/2 mb-3 z-20 bg-[#18122B]/95 p-2 rounded flex flex-col items-center border border-[#605985] shadow-lg">
                    <input
                        type="range"
                        min="0"
                        max="1"
                        step="0.01"
                        class="h-24 w-2 accent-[#605985] bg-[#393053] rounded appearance-none volume-vertical"
                        value={volume}
                        on:input={handleVolumeChange}
                    />
                </div>
                {/if}
            </div>
            <button on:click={handleFullscreen} class="hover:bg-[#443C68]/80 p-2 rounded-full transition duration-200">
                <Icon color="#fef9ec" src={isFullscreen ? RiMediaFullscreenExitFill : RiMediaFullscreenLine} size="22px" />
            </button>
          </div>
        </div>
      {/if}
    </div>
</div>

<style>
  .hide-cursor {
    cursor: none !important;
  }
  @keyframes fade-in {
    from { opacity: 0; transform: translateY(10px); }
    to { opacity: 1; transform: translateY(0); }
  }
  @keyframes fade-out {
    from { opacity: 1; transform: translateY(0); }
    to { opacity: 0; transform: translateY(10px); }
  }
  .animate-fade-in {
    animation: fade-in 0.2s ease;
  }
  .animate-fade-out {
    animation: fade-out 0.3s ease forwards;
  }
  input[type="range"].volume-vertical {
    writing-mode: bt-lr;
    -webkit-appearance: slider-vertical;
    appearance: slider-vertical;
    width: 2rem;
    height: 6rem;
    padding: 0;
    margin: 0;
  }
  .group:hover .animate-fade-in, .group:focus-within .animate-fade-in {
    opacity: 1;
  }
  /* Mini player and mobile styles removed */
</style>
<script lang="ts">
  //Use "npm i" in your terminal
  //Use "npm run dev" in your terminal
  //Set 'isPlaying' to true
  //Click the Button
  let isPlaying = true;
</script>

<div>
  <ul>
    <li>Use "npm i" in your terminal</li>
    <li>Use "npm run dev" in your terminal</li>
    <li>Set 'isPlaying' at the top to true</li>
  </ul>
</div>
{#if isPlaying}
  <p>Congrats you can delete all this code</p>
{:else}
  <p>Follow the instructions above</p>
{/if}
<p class="text-8xl {isPlaying ? 'text-blue-500' : 'text-red-500'}  font-bold">
  THIS TEXT SHOULD BE BIG, RED AND BOLD IF false AND BLUE IF true
</p>

<button
  class="bg-black text-white p-4"
  on:click={() => (isPlaying = !isPlaying)}>Click Here</button
>

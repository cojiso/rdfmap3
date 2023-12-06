<script>
// @ts-ignore
import RangeSlider from 'svelte-range-slider-pips';
// let values = [20,80,100,0];
// let values = [50];
let values = [1960,1990];
</script>

<h1>Welcome to SvelteKit</h1>
<p>Visit <a href="https://kit.svelte.dev">kit.svelte.dev</a> to read the documentation</p>

<!-- <RangeSlider range pips bind:values class="sliders" /> -->
<RangeSlider range float pips pipstep={10} first="label" rest="label" min={1900} max={2023} bind:values />
<!-- <RangeSlider class="sliderd" range pushy pips bind:values /> -->
<p>{values}</p>
    
<style>
:global(.range) {
    width: auto;
    overflow: visible;
}
:global(.slider)  {
    z-index: 1;
}
</style>
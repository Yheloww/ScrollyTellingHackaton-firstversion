<script>
	import { themes } from '../config.js';
	import { getContext } from 'svelte';

	export let theme = getContext('theme');
</script>

<section style="color: {themes[theme]['text']}; background-color: {themes[theme]['background']}">
	<div class="col-medium">
		<slot></slot>
	</div>
</section>

<style>

	section{
		display: block;
  margin-left: auto;
  margin-right: auto;
  width: 100%;
  overflow: hidden;
  background-color: #F9F8F4;
	}
</style>
<script>
	import { AlertCircle } from '@lucide/svelte'
	import { onMount } from 'svelte'
	// Check if we're in browser
	const browser = typeof window !== 'undefined'

	let { errors, title = 'Form problem' } = $props()
	let errorContainer = $state(null)
	
	// Focus the error container when it appears
	onMount(() => {
		if (browser && errors?._errors?.length && errorContainer) {
			// Use a slight delay to ensure screen readers announce the alert
			setTimeout(() => {
				errorContainer.focus()
			}, 100)
		}
	})
</script>

{#if errors?._errors?.length}
	<div 
		class="form-error" 
		role="alert" 
		aria-live="assertive"
		tabindex="-1"
		bind:this={errorContainer}
	>
		<div class="form-error__icon-wrapper" aria-hidden="true">
			<AlertCircle size={16} />
		</div>
		<strong class="form-error__title" id="error-heading">{ title }</strong>
		<ul class="form-error__list" aria-labelledby="error-heading">
			{#each errors._errors as error, index}
				<li class="form-error__item" id="error-item-{index}">{ error }</li>
			{/each}
		</ul>
	</div>
{/if}

<style>
	@import './FormErrors.scss';
</style>

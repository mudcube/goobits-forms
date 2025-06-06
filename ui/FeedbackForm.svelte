<script>
	import { browser } from '$app/environment'
	import { Loader2 } from '@lucide/svelte'
	import { onMount } from 'svelte'
	import { z } from 'zod'

	// Import shared form service functions
	import {
		createFormSubmitHandler,
		handleFieldInput,
		handleFieldTouch,
		initializeForm,
		initializeFormState,
		resetForm as resetFormService
	} from '../services/formService.js'
	
	// Import logger utility
	import { createLogger } from '../utils/logger.js'
	
	// Import message helpers
	import { createMessageGetter } from '../utils/messages.js'
	import { defaultMessages } from '../config/defaultMessages.js'
	
	const logger = createLogger('FeedbackForm')
	

	// Props
	const props = $props()
	
	const defaultProps = {
		feedbackType: '',
		userComment: '',
		userName: '',
		userEmail: '',
		messages: {}
	}
	
	// Create message getter
	const getMessage = createMessageGetter({ ...defaultMessages, ...props.messages })

	// Define validation schema using zod
	const feedbackSchema = z.object({
		feedbackType: z.string().min(1, 'Please select whether the page was helpful or not.'),
		userComment: z.string().min(1, 'Please share your thoughts about this page.'),
		userEmail: z.string().email('Please enter a valid email address').optional().or(z.literal('')),
		userName: z.string().optional()
	})

	// Initialize form state using shared service
	const formState = initializeFormState({
		isFormVisible: props.isFormVisible || false,
		isThankYouVisible: props.isThankYouVisible || false,
		currentPagePath: '',
		touched: {}
	})

	// UI State
	let currentPagePath = $state(formState.currentPagePath)
	let isFormVisible = $state(formState.isFormVisible)
	let isThankYouVisible = $state(formState.isThankYouVisible)
	let recaptcha = $state(formState.recaptcha)
	let submissionError = $state(formState.submissionError)
	let submitting = $state(false)
	let touched = $state(formState.touched)
	let thankYouRef = $state(null)

	// Define the submit handler using shared function
	const handleSubmit = async(formData) => {
		const submitHandler = createFormSubmitHandler({
			validateForm: () => !Object.values($errors).some(v => v),
			recaptcha,
			prepareFormData: async(formData, recaptchaToken) => {
				const currentPage = page?.url?.pathname || 'Unknown page'

				return {
					name: formData.get('userName') || 'Page Visitor',
					email: formData.get('userEmail') || 'no-reply@example.com',
					category: 'product-feedback',
					message: formData.get('userComment'),
					recaptchaToken,
					featureArea: 'Page Feedback',
					coppa: true,
					page: currentPage,
					helpful: formData.get('feedbackType') === 'yes' ? 'Yes ✓' : 'No ✗'
				}
			},
			submitForm: async(formDataToSubmit) => {
				submitting = true
				return await submitContactForm(formDataToSubmit)
			},
			onSuccess: () => {
				resetForm()
				isFormVisible = false
				isThankYouVisible = true
				
				// Focus the thank you message when it appears
				if (browser && thankYouRef) {
					setTimeout(() => {
						thankYouRef.focus()
					}, 100)
				}
			},
			onError: (error) => {
				submissionError = error
			}
		})

		await submitHandler(formData)
	}

	// Initialize form with shared service
	const form = initializeForm({
		initialData: defaultProps,
		schema: feedbackSchema,
		onSubmitHandler: handleSubmit,
		extraOptions: {
			onError: ({ result }) => {
				// Handle validation errors from server
				if (result?.error) {
					submissionError = result.error
				}
			}
		}
	})

	const { form: formData, errors, enhance, validate } = form

	// Effect to reset form on page change
	$effect(() => {
		if (page && page.url && page.url.pathname !== currentPagePath) {
			currentPagePath = page.url.pathname
			resetForm()

		}
	})

	// Initialize on mount with shared lifecycle setup
	onMount(async() => {
		resetForm()

		if (page && page.url) {
			currentPagePath = page.url.pathname
		}

		// ReCAPTCHA setup would go here if needed
	})

	// Private functions

	/**
	 * Show the feedback form
	 */
	function showFeedbackForm() {
		isFormVisible = true
	}

	/**
	 * Reset the form to its initial state
	 */
	function resetForm() {
		isFormVisible = false
		isThankYouVisible = false

		// Use the shared reset function
		resetFormService(
			formData.set,
			defaultProps,
			{ submissionError, touched }
		)
	}

	/**
	 * Handle field blur event using shared function
	 * @param {string} fieldName
	 */
	function handleBlur(fieldName) {
		touched = handleFieldTouch(touched, fieldName)
	}

	/**
	 * Handle field input event using shared function
	 * @param {string} fieldName
	 */
	function handleInput(fieldName) {
		handleFieldInput(touched, fieldName, validate)
	}
</script>

<div class="feedback">
	{#if isThankYouVisible}
		<div 
			class="feedback__thank-you" 
			role="status" 
			aria-live="polite" 
			tabindex="-1" 
			bind:this={thankYouRef}
		>
			<strong>Thank you for your feedback!</strong>
			<div class="feedback__actions feedback__actions--thank-you">
				<button class="feedback__btn feedback__btn--secondary" onclick={resetForm}>Close</button>
			</div>
		</div>
	{:else}
		<div class="feedback__content">
			{#if !isFormVisible}
				<p class="feedback__prompt">
					<i>Was this help page useful?</i>
					<button class="feedback__trigger" onclick={showFeedbackForm}>
						<span>Send feedback</span>
					</button>
				</p>
			{:else}
				<form class="feedback__form" method="POST" use:enhance onreset={resetForm} name="feedback">
					<h3 class="feedback__title">Was this help page useful?</h3>

					{#if submissionError}
						<div class="feedback__error" role="alert">
							<div class="feedback__error-message">{submissionError}</div>
						</div>
					{/if}

					<div class="feedback__choices">
						<div class="feedback__field">
							<div class="feedback__radio-group">
								<label class="feedback__radio" data-fs-field-errors={$errors.feedbackType ? 'feedbackType' : ''}>
									<input
										bind:group={$formData.feedbackType}
										name="feedbackType"
										onblur={() => handleBlur('feedbackType')}
										oninput={() => handleInput('feedbackType')}
										type="radio"
										value="yes"
									>
									<span class="feedback__radio-text">Yes</span>
								</label>
								<label class="feedback__radio" data-fs-field-errors={$errors.feedbackType ? 'feedbackType' : ''}>
									<input
										bind:group={$formData.feedbackType}
										name="feedbackType"
										onblur={() => handleBlur('feedbackType')}
										oninput={() => handleInput('feedbackType')}
										type="radio"
										value="no"
									>
									<span class="feedback__radio-text">No</span>
								</label>
							</div>
							{#if $errors.feedbackType && touched.feedbackType}
								<div class="feedback__field-error" data-fs-field-error="feedbackType">{$errors.feedbackType}</div>
							{/if}
						</div>
					</div>

					<label class="feedback__field">
						<span class="feedback__label">Your Name (Optional)</span>
						<input
							bind:value={$formData.userName}
							class="feedback__input"
							name="userName"
							onblur={() => handleBlur('userName')}
							oninput={() => handleInput('userName')}
							type="text"
						>
					</label>

					<label class="feedback__field">
						<span class="feedback__label">Your Thoughts</span>
						<textarea
							bind:value={$formData.userComment}
							class:feedback__textarea--error={$errors.userComment && touched.userComment}
							class="feedback__textarea"
							data-fs-field-errors={$errors.userComment ? 'userComment' : ''}
							name="userComment"
							onblur={() => handleBlur('userComment')}
							oninput={() => handleInput('userComment')}
						></textarea>
						{#if $errors.userComment && touched.userComment}
							<div class="feedback__field-error" data-fs-field-error="userComment">{$errors.userComment}</div>
						{/if}
					</label>

					<label class="feedback__field">
						<span class="feedback__label">Email (If you'd like a reply)</span>
						<input
							bind:value={$formData.userEmail}
							class="feedback__input"
							data-fs-field-errors={$errors.userEmail ? 'userEmail' : ''}
							name="userEmail"
							onblur={() => handleBlur('userEmail')}
							oninput={() => handleInput('userEmail')}
							type="email"
						>
						{#if $errors.userEmail && touched.userEmail}
							<div class="feedback__field-error" data-fs-field-error="userEmail">{$errors.userEmail}</div>
						{/if}
					</label>

					<p class="feedback__note">
						Your input helps us improve our website for everyone.
					</p>

					<div class="feedback__actions">
						{#if !submitting}
							<button type="reset" class="feedback__btn feedback__btn--secondary">Cancel</button>
						{/if}
						<button type="submit" class="feedback__btn feedback__btn--primary" disabled={submitting}>
							{#if submitting}
								<Loader2 class="animate-spin" size={18} />
								<span>Sending...</span>
							{:else}
								<span>Send feedback</span>
							{/if}
						</button>
					</div>
				</form>
			{/if}
		</div>
	{/if}
</div>

<style>
	@import './FeedbackForm.scss';
</style>
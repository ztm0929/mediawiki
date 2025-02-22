<template>
	<cdx-field
		class="mw-block-fieldset"
		:is-fieldset="true"
		:disabled="formDisabled"
	>
		<div ref="messagesContainer" class="mw-block-messages">
			<cdx-message
				v-if="success"
				type="success"
				:allow-user-dismiss="true"
				class="mw-block-success"
			>
				<p><strong>{{ $i18n( 'blockipsuccesssub' ) }}</strong></p>
				<!-- eslint-disable-next-line vue/no-v-html -->
				<p v-html="$i18n( 'block-success', store.targetUser ).parse()"></p>
			</cdx-message>
			<cdx-message
				v-for="( formError, index ) in formErrors"
				:key="index"
				type="error"
				class="mw-block-error"
				inline
			>
				{{ formError }}
			</cdx-message>
		</div>
		<user-lookup
			v-model="store.targetUser"
			:form-submitted="formSubmitted"
			@input="store.alreadyBlocked = false"
		></user-lookup>
		<block-log
			v-if="blockEnableMultiblocks"
			:key="`${submitCount}-active`"
			block-log-type="active"
		></block-log>
		<block-log
			:key="`${submitCount}-recent`"
			block-log-type="recent"
		></block-log>
		<block-log
			v-if="blockShowSuppressLog"
			:key="`${submitCount}-suppress`"
			block-log-type="suppress"
		></block-log>
		<block-type-field></block-type-field>
		<expiry-field :form-submitted="formSubmitted"></expiry-field>
		<reason-field
			v-model:selected="store.reason"
			v-model:other="store.reasonOther"
		></reason-field>
		<block-details-field>
		</block-details-field>
		<additional-details-field>
		</additional-details-field>
		<cdx-field v-if="store.confirmationRequired">
			<cdx-checkbox
				v-model="store.confirmationChecked"
				class="mw-block-confirm"
			>
				{{ $i18n( 'ipb-confirm' ).text() }}
			</cdx-checkbox>
		</cdx-field>
		<hr class="mw-block-hr">
		<cdx-button
			action="destructive"
			weight="primary"
			class="mw-block-submit"
			:disabled="formDisabled || ( store.confirmationRequired && !store.confirmationChecked )"
			@click="handleSubmit"
		>
			{{ submitButtonMessage }}
		</cdx-button>
	</cdx-field>
</template>

<script>
const { computed, defineComponent, nextTick, ref, Ref, watch } = require( 'vue' );
const { storeToRefs } = require( 'pinia' );
const { CdxButton, CdxCheckbox, CdxField, CdxMessage } = require( '@wikimedia/codex' );
const useBlockStore = require( './stores/block.js' );
const UserLookup = require( './components/UserLookup.vue' );
const BlockLog = require( './components/BlockLog.vue' );
const BlockTypeField = require( './components/BlockTypeField.vue' );
const ExpiryField = require( './components/ExpiryField.vue' );
const ReasonField = require( './components/ReasonField.vue' );
const BlockDetailsField = require( './components/BlockDetailsField.vue' );
const AdditionalDetailsField = require( './components/AdditionalDetailsField.vue' );

module.exports = exports = defineComponent( {
	name: 'SpecialBlock',
	components: {
		UserLookup,
		BlockLog,
		BlockTypeField,
		ExpiryField,
		ReasonField,
		BlockDetailsField,
		AdditionalDetailsField,
		CdxButton,
		CdxCheckbox,
		CdxField,
		CdxMessage
	},
	setup() {
		const store = useBlockStore();
		const blockEnableMultiblocks = mw.config.get( 'blockEnableMultiblocks' ) || false;
		const blockShowSuppressLog = mw.config.get( 'blockShowSuppressLog' ) || false;
		const success = ref( false );
		/**
		 * Whether the form has been submitted. This is used to only show error states
		 * after the user has attempted to submit the form.
		 *
		 * @type {Ref<boolean>}
		 */
		const formSubmitted = ref( false );
		const formDisabled = ref( false );
		const messagesContainer = ref();
		// Value to use for BlockLog component keys, so they reload after saving.
		const submitCount = ref( 0 );
		// eslint-disable-next-line arrow-body-style
		const submitButtonMessage = computed( () => {
			return mw.message( store.alreadyBlocked ? 'ipb-change-block' : 'ipbsubmit' ).text();
		} );

		// Show an error message if the target user is the current user.
		const { formErrors, targetUser } = storeToRefs( store );
		watch( targetUser, ( newValue ) => {
			if ( newValue === mw.config.get( 'wgUserName' ) ) {
				formErrors.value.push( mw.msg( 'ipb-blockingself', 'ipb-confirmaction' ) );
			} else {
				formErrors.value = [];
			}
			success.value = false;
		} );

		/**
		 * Handle form submission. If the form is invalid, show the browser's
		 * validation messages along with any custom validation messages, and
		 * scroll to the first error message to ensure the user sees it.
		 *
		 * If the form is valid, send the block request to the server,
		 * add the success message and scroll to it.
		 *
		 * @param {Event} event
		 */
		function handleSubmit( event ) {
			event.preventDefault();
			formSubmitted.value = true;
			success.value = false;

			// checkValidity() executes browser form validation, which triggers automatic
			// validation states on applicable components (e.g. fields with `required` attr).
			if ( event.target.form.checkValidity() && store.expiry ) {
				store.doBlock()
					.done( () => {
						success.value = true;
						formErrors.value = [];
						formSubmitted.value = false;
					} )
					.fail( ( _, errorObj ) => {
						formErrors.value = [ errorObj.error.info ];
						success.value = false;
					} )
					.always( () => {
						formDisabled.value = false;
						messagesContainer.value.scrollIntoView( { behavior: 'smooth' } );
						if ( success.value ) {
							// Bump the submitCount (to re-render the logs) after scrolling
							// because the log tables may change the length of the page.
							submitCount.value++;
						}
					} );
			} else {
				// nextTick() needed to ensure error messages are rendered before scrolling.
				nextTick( () => {
					// Currently, only the expiry field has custom validations.
					// Scrolling to `cdx-message--error` is merely future-proofing to
					// ensure the user sees the error message, wherever it may be.
					// Actual validation logic should live in the respective component,
					// and only apply after the form has been submitted via a `formSubmitted` prop.
					document.querySelector( '.cdx-message--error' )
						.scrollIntoView( { behavior: 'smooth' } );
				} );
			}
		}

		return {
			store,
			formSubmitted,
			formDisabled,
			messagesContainer,
			formErrors,
			success,
			submitCount,
			submitButtonMessage,
			handleSubmit,
			blockEnableMultiblocks,
			blockShowSuppressLog
		};
	}
} );
</script>

<style lang="less">
@import 'mediawiki.skin.variables.less';

// HACK: The accordions within the form need to be full width.
.mw-htmlform-codex {
	max-width: unset;
}

// HACK: Set the max-width of the fields back to what they should be.
.cdx-field:not( .mw-block-fieldset ) {
	max-width: @size-4000;
}

.mw-block-hideuser,
.mw-block-confirm {
	.cdx-checkbox__label .cdx-label__label__text {
		font-weight: @font-weight-bold;
	}
}

.mw-block-hr {
	margin-top: @spacing-200;
}

.mw-block-submit.cdx-button {
	margin-top: @spacing-100;
}

.mw-block-error {
	margin-left: @spacing-75;
}

// Hide the log showing at the bottom of page.
.mw-warning-with-logexcerpt {
	display: none;
}
</style>

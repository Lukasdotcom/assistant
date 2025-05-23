<!--
  - SPDX-FileCopyrightText: 2024 Nextcloud GmbH and Nextcloud contributors
  - SPDX-License-Identifier: AGPL-3.0-or-later
-->
<template>
	<div
		class="message"
		@mouseover="showMessageActions = true"
		@mouseleave="showMessageActions = false">
		<MessageActions v-show="showMessageActions"
			class="message__actions"
			:show-regenerate="showRegenerate"
			:delete-loading="deleteLoading"
			:regenerate-loading="regenerateLoading"
			@copy="copyMessage(message.content)"
			@regenerate="$emit('regenerate')"
			@delete="$emit('delete')" />
		<div class="message__header">
			<div class="message__header__role">
				<NcAvatar
					:user="message.role === 'human' ? userId : 'Nextcloud Assistant'"
					:display-name="message.role === 'human' ? displayName : t('assistant', 'Nextcloud Assistant')"
					:is-no-user="message.role === 'assistant'"
					:show-user-status="false">
					<template #icon>
						<NcLoadingIcon v-if="message.role === 'human' && newMessageLoading" :size="20" />
						<AssistantIcon v-else-if="message.role === 'assistant'" :size="20" />
					</template>
				</NcAvatar>
				<div class="message__header__role__name">
					{{ message.role === 'human' ? displayName : t('assistant', 'Nextcloud Assistant') }}
				</div>
			</div>
			<NcDateTime class="message__header__timestamp" :timestamp="new Date((message?.timestamp ?? 0) * 1000)" :ignore-seconds="true" />
		</div>
		<NcRichText class="message__content"
			:text="message.content || t('assistant', '(empty message)')"
			:use-markdown="true"
			:reference-limit="1"
			:references="references"
			:autolink="true" />
	</div>
</template>

<script>
import AssistantIcon from '../icons/AssistantIcon.vue'

import NcAvatar from '@nextcloud/vue/dist/Components/NcAvatar.js'
import NcDateTime from '@nextcloud/vue/dist/Components/NcDateTime.js'
import NcLoadingIcon from '@nextcloud/vue/dist/Components/NcLoadingIcon.js'
import { NcRichText } from '@nextcloud/vue/dist/Components/NcRichText.js'

import MessageActions from './MessageActions.vue'

import { getCurrentUser } from '@nextcloud/auth'
import { showSuccess } from '@nextcloud/dialogs'
import { generateOcsUrl } from '@nextcloud/router'
import axios from '@nextcloud/axios'

const PLAIN_URL_PATTERN = /(?:\s|^|\()((?:https?:\/\/)(?:[-A-Z0-9+_.]+(?::[0-9]+)?(?:\/[-A-Z0-9+&@#%?=~_|!:,.;()]*)*))(?:\s|$|\))/ig
const MARKDOWN_LINK_PATTERN = /\[[-A-Z0-9+&@#%?=~_|!:,.;()]+\]\(((?:https?:\/\/)(?:[-A-Z0-9+_.]+(?::[0-9]+)?(?:\/[-A-Z0-9+&@#%?=~_|!:,.;]*)*))\)/ig

export default {
	name: 'Message',

	components: {
		AssistantIcon,

		NcAvatar,
		NcDateTime,
		NcLoadingIcon,
		NcRichText,

		MessageActions,
	},

	props: {
		// { id: number, session_id: number, role: string, content: string, timestamp: number }
		message: {
			type: Object,
			required: true,
		},
		showRegenerate: {
			type: Boolean,
			default: false,
		},
		deleteLoading: {
			type: Boolean,
			default: false,
		},
		regenerateLoading: {
			type: Boolean,
			default: false,
		},
		newMessageLoading: {
			type: Boolean,
			default: false,
		},
	},

	data: () => {
		return {
			displayName: getCurrentUser()?.displayName ?? getCurrentUser()?.uid ?? t('assistant', 'You'),
			userId: getCurrentUser()?.uid ?? t('assistant', 'You'),
			showMessageActions: false,
			// we could initialize this with undefined but then NcRichText will try to find links
			// and resolve them before we had a chance to do so, resulting in unnecessary requests to references/resolve
			// with [] we inhibit the extract+resolve mechanism on NcRichText
			// TODO This can be removed (and all the custom extract+resolve logic) when fixed in NcRichText
			references: [],
		}
	},

	mounted() {
		this.fetch()
	},

	methods: {
		copyMessage(message) {
			navigator.clipboard.writeText(message)
			showSuccess(t('assistant', 'Message copied to clipboard'))
		},
		fetch() {
			const urlMatch = (new RegExp(PLAIN_URL_PATTERN).exec(this.message.content.trim()))
			const mdMatch = (new RegExp(MARKDOWN_LINK_PATTERN).exec(this.message.content.trim()))
			const firstMatch = urlMatch
				? urlMatch[1].replaceAll(/[).,:;!?]+$/g, '')
				: mdMatch
					? mdMatch[1]
					: false
			if (firstMatch) {
				axios.get(generateOcsUrl('references/resolve') + `?reference=${encodeURIComponent(firstMatch)}`)
					.then((response) => {
						this.references = Object.values(response.data.ocs.data.references)
					})
					.catch((error) => {
						console.error('Failed to extract references', error)
					})
			}
		},
	},
}
</script>

<style lang="scss" scoped>
.message {
	border-radius: var(--border-radius-large);
	padding: 0.5em;
	position: relative;

	&:hover {
		background-color: var(--color-background-hover);
	}

	&__header {
		display: flex;
		flex-direction: row;
		align-items: center;
		justify-content: space-between;

		&__role {
			display: flex;
			flex-direction: row;
			align-items: center;
			gap: 0.5em;

			&__name {
				font-weight: bold;
			}
		}

		&__timestamp {
			color: var(--color-text-maxcontrast);
		}
	}

	&__content {
		margin-left: 2.6em;
		overflow: auto;

		:deep ol {
			margin-left: 1em;
		}

		:deep .widget-default, :deep .widget-custom {
			width: auto !important;
		}
	}
}
</style>

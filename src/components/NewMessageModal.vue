<template>
	<Modal
		size="large"
		:title="t('mail', 'New message')"
		@close="$emit('close')">
		<Composer
			:from-account="composerData.accountId"
			:to="composerData.to"
			:cc="composerData.cc"
			:bcc="composerData.bcc"
			:subject="composerData.subject"
			:body="composerData.body"
			:draft="saveDraft"
			:send="sendMessage"
			:reply-to="composerData.replyTo"
			:forward-from="composerData.forwardFrom" />
	</Modal>
</template>
<script>
import Modal from '@nextcloud/vue/dist/Components/Modal'
import Composer from './Composer'
import logger from '../logger'
import { detect, html, plain, toPlain } from '../util/text'
import { saveDraft, sendMessage } from '../service/MessageService'
import { buildForwardSubject, buildRecipients as buildReplyRecipients, buildReplySubject } from '../ReplyBuilder'
import { showWarning } from '@nextcloud/dialogs'
import { translate as t } from '@nextcloud/l10n'
export default {
	name: 'NewMessageModal',
	components: {
		Modal,
		Composer,
	},
	data() {
		return {
			showModal: false,
			draft: undefined,
			displayNewMessage: false,
		}
	},
	computed: {
		newMessage() {
			return (
				this.$route.params.threadId === 'new'
				|| this.$route.params.threadId === 'reply'
				|| this.$route.params.threadId === 'replyAll'
				|| this.$route.params.threadId === 'asNew'
			)
		},
		composerData() {
			if (this.draft !== undefined) {
				logger.info('todo: handle draft data', { draft: this.draft })
				return {
					to: this.draft.to,
					cc: this.draft.cc,
					bcc: this.draft.bcc,
					subject: this.draft.subject,
					body: this.draft.hasHtmlBody ? html(this.draft.body) : plain(this.draft.body),
				}
			} else if (this.$route.query.messageId !== undefined) {
				// Forward or reply to a message
				const message = this.original
				logger.debug('forwarding or replying to message', { message })

				if (this.$route.params.threadId === 'reply') {
					logger.debug('simple reply', {
						message,
					})

					return {
						accountId: message.accountId,
						to: message.from,
						cc: [],
						subject: buildReplySubject(message.subject),
						body: this.originalBody,
						originalBody: this.originalBody,
						replyTo: message,
					}
				} else if (this.$route.params.threadId === 'replyAll') {
					logger.debug('replying to all', { original: this.original })
					const account = this.$store.getters.getAccount(message.accountId)
					const recipients = buildReplyRecipients(message, {
						email: account.emailAddress,
						label: account.name,
					})

					return {
						accountId: message.accountId,
						to: recipients.to,
						cc: recipients.cc,
						subject: buildReplySubject(message.subject),
						body: this.originalBody,
						originalBody: this.originalBody,
						replyTo: message,
					}
				} else if (this.$route.params.threadId === 'asNew') {
					logger.debug('composing as new', { original: this.original })

					if (this.original.attachments.length) {
						showWarning(t('mail', 'Attachments were not copied. Please add them manually.'))
					}

					return {
						accountId: message.accountId,
						to: message.to,
						cc: message.cc,
						subject: message.subject,
						body: this.originalBody,
						originalBody: this.originalBody,
					}
				} else {
					// forward
					return {
						accountId: message.accountId,
						to: [],
						cc: [],
						subject: buildForwardSubject(message.subject),
						body: this.originalBody,
						originalBody: this.originalBody,
						forwardFrom: message,
					}
				}
			} else {
				// New or mailto: message
				logger.debug('composing a new message or handling a mailto link', {
					threadId: this.$route.params.threadId,
				})

				let accountId
				// Only preselect an account when we're not in a unified mailbox
				if (this.$route.params.accountId !== 0 && this.$route.params.accountId !== '0') {
					accountId = parseInt(this.$route.params.accountId, 10)
				}

				return {
					accountId,
					to: this.stringToRecipients(this.$route.query.to),
					cc: this.stringToRecipients(this.$route.query.cc),
					subject: this.$route.query.subject || '',
					body: this.$route.query.body ? detect(this.$route.query.body) : html(''),
				}
			}
		},
	},
	methods: {
		stringToRecipients(str) {
			if (str === undefined) {
				return []
			}

			return [
				{
					label: str,
					email: str,
				},
			]
		},
		async saveDraft(data) {
			if (data.draftId === undefined && this.draft) {
				logger.debug('draft data does not have a draftId, adding one', { draft: this.draft, data, id: this.draft.databaseId })
				data.draftId = this.draft.databaseId
			}
			const dataForServer = {
				...data,
				body: data.isHtml ? data.body.value : toPlain(data.body).value,
			}
			const { id } = await saveDraft(data.account, dataForServer)

			// Remove old draft envelope
			this.$store.commit('removeEnvelope', { id: data.draftId })
			this.$store.commit('removeMessage', { id: data.draftId })

			// Fetch new draft envelope
			await this.$store.dispatch('fetchEnvelope', id)

			// Update route to new draft (actual redirect will be skipped)
			const account = this.$store.getters.getAccount(data.account)
			if (parseInt(this.$route.params.mailboxId, 10) === account.draftsMailboxId) {
				this.newDraftId = id
				this.$router.replace({
					to: 'message',
					params: {
						mailboxId: this.$route.params.mailboxId,
						threadId: this.$route.params.threadId,
						draftId: id,
					},
				})
			}

			return id
		},
		async sendMessage(data) {
			logger.debug('sending message', { data })
			const dataForServer = {
				...data,
				body: data.isHtml ? data.body.value : toPlain(data.body).value,
			}
			await sendMessage(data.account, dataForServer)

			// Remove old draft envelope
			this.$store.commit('removeEnvelope', { id: data.draftId })
			this.$store.commit('removeMessage', { id: data.draftId })
		},
	},
}
</script>

<style lang="scss" scoped>
::v-deep .modal-container {
	min-width: 50%;
}
</style>

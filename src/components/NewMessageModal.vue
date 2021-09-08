<template>
	<Modal
		size="large"
		:title="t('mail', 'New message')"
		@close="$emit('close')">
		<NewMessageDetail />
	</Modal>
</template>
<script>
import Modal from '@nextcloud/vue/dist/Components/Modal'
import NewMessageDetail from './NewMessageDetail'
export default {
	name: 'NewMessageModal',
	components: {
		Modal,
		NewMessageDetail,
	},
}
</script>

<style lang="scss" scoped>
@media only screen and (max-width: 600px) {
	::v-deep .modal-container {
		max-width: 80%;
	}
}
::v-deep .modal-container {
	width: 80%;
}
::v-deep .modal-wrapper .modal-container {
	overflow-y: auto;
	overflow-x: auto;
}
</style>

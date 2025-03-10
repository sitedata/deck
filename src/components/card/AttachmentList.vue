<!--
  - @copyright Copyright (c) 2020 Jakob Röhrl <jakob.roehrl@web.de>
  -
  - @author Jakob Röhrl <jakob.roehrl@web.de>
  -
  - @license GNU AGPL version 3 or any later version
  -
  - This program is free software: you can redistribute it and/or modify
  - it under the terms of the GNU Affero General Public License as
  - published by the Free Software Foundation, either version 3 of the
  - License, or (at your option) any later version.
  -
  - This program is distributed in the hope that it will be useful,
  - but WITHOUT ANY WARRANTY; without even the implied warranty of
  - MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
  - GNU Affero General Public License for more details.
  -
  - You should have received a copy of the GNU Affero General Public License
  - along with this program. If not, see <http://www.gnu.org/licenses/>.
  -
  -->

<template>
	<AttachmentDragAndDrop :card-id="cardId" class="drop-upload--sidebar">
		<div class="button-group">
			<button class="icon-upload" @click="uploadNewFile()">
				{{ t('deck', 'Upload new files') }}
			</button>
			<button class="icon-folder" @click="shareFromFiles()">
				{{ t('deck', 'Share from Files') }}
			</button>
		</div>
		<input ref="filesAttachment"
			type="file"
			style="display: none;"
			multiple
			@change="handleUploadFile">
		<ul class="attachment-list">
			<li v-for="attachment in uploadQueue" :key="attachment.name" class="attachment">
				<a class="fileicon" :style="mimetypeForAttachment()" />
				<div class="details">
					<a>
						<div class="filename">
							<span class="basename">{{ attachment.name }}</span>
						</div>
						<progress :value="attachment.progress" max="100" />
					</a>
				</div>
			</li>
			<li v-for="attachment in attachments"
				:key="attachment.id"
				class="attachment">
				<a class="fileicon"
					:style="mimetypeForAttachment(attachment)"
					@click.prevent="showViewer(attachment)" />
				<div class="details">
					<a @click.prevent="showViewer(attachment)">
						<div class="filename">
							<span class="basename">{{ attachment.data }}</span>
						</div>
						<span class="filesize">{{ formattedFileSize(attachment.extendedData.filesize) }}</span>
						<span class="filedate">{{ relativeDate(attachment.createdAt*1000) }}</span>
						<span class="filedate">{{ attachment.createdBy }}</span>
					</a>
				</div>
				<Actions v-if="selectable">
					<ActionButton icon="icon-confirm" @click="$emit('selectAttachment', attachment)">
						{{ t('deck', 'Add this attachment') }}
					</ActionButton>
				</Actions>
				<Actions v-if="removable" :force-menu="true">
					<ActionLink v-if="attachment.extendedData.fileid" icon="icon-folder" :href="internalLink(attachment)">
						{{ t('deck', 'Show in Files') }}
					</ActionLink>
					<ActionButton v-if="attachment.extendedData.fileid" icon="icon-delete" @click="unshareAttachment(attachment)">
						{{ t('deck', 'Unshare file') }}
					</ActionButton>

					<ActionButton v-if="!attachment.extendedData.fileid && attachment.deletedAt === 0" icon="icon-delete" @click="$emit('deleteAttachment', attachment)">
						{{ t('deck', 'Delete Attachment') }}
					</ActionButton>
					<ActionButton v-else-if="!attachment.extendedData.fileid" icon="icon-history" @click="$emit('restoreAttachment', attachment)">
						{{ t('deck', 'Restore Attachment') }}
					</ActionButton>
				</Actions>
			</li>
		</ul>
	</AttachmentDragAndDrop>
</template>

<script>
import axios from '@nextcloud/axios'
import { Actions, ActionButton, ActionLink } from '@nextcloud/vue'
import AttachmentDragAndDrop from '../AttachmentDragAndDrop'
import relativeDate from '../../mixins/relativeDate'
import { formatFileSize } from '@nextcloud/files'
import { generateUrl, generateOcsUrl } from '@nextcloud/router'
import { mapState } from 'vuex'
import { loadState } from '@nextcloud/initial-state'
import attachmentUpload from '../../mixins/attachmentUpload'
import { getFilePickerBuilder } from '@nextcloud/dialogs'
const maxUploadSizeState = loadState('deck', 'maxUploadSize')

const picker = getFilePickerBuilder(t('deck', 'File to share'))
	.setMultiSelect(false)
	.setModal(true)
	.setType(1)
	.allowDirectories()
	.build()

export default {
	name: 'AttachmentList',
	components: {
		Actions,
		ActionButton,
		ActionLink,
		AttachmentDragAndDrop,
	},
	mixins: [relativeDate, attachmentUpload],

	props: {
		cardId: {
			type: Number,
			required: true,
		},
		selectable: {
			type: Boolean,
			required: false,
		},
		removable: {
			type: Boolean,
			required: false,
		},
	},
	data() {
		return {
			modalShow: false,
			file: '',
			overwriteAttachment: null,
			isDraggingOver: false,
			maxUploadSize: maxUploadSizeState,
		}
	},
	computed: {
		attachments() {
			return [...this.$store.getters.attachmentsByCard(this.cardId)].filter(attachment => attachment.deletedAt >= 0).sort((a, b) => b.id - a.id)
		},
		mimetypeForAttachment() {
			return (attachment) => {
				if (!attachment) {
					return {}
				}
				const url = attachment.extendedData.hasPreview ? this.attachmentPreview(attachment) : OC.MimeType.getIconUrl(attachment.extendedData.mimetype)
				const styles = {
					'background-image': `url("${url}")`,
				}
				return styles
			}
		},
		attachmentPreview() {
			return (attachment) => (attachment.extendedData.fileid ? generateUrl(`/core/preview?fileId=${attachment.extendedData.fileid}&x=64&y=64&a=true`) : null)
		},
		attachmentUrl() {
			return (attachment) => generateUrl(`/apps/deck/cards/${attachment.cardId}/attachment/${attachment.id}`)
		},
		internalLink() {
			return (attachment) => generateUrl('/f/' + attachment.extendedData.fileid)
		},
		formattedFileSize() {
			return (filesize) => formatFileSize(filesize)
		},
		...mapState({
			currentBoard: state => state.currentBoard,
		}),
		isReadOnly() {
			return !this.$store.getters.canEdit
		},
		dropHintText() {
			if (this.isReadOnly) {
				return t('deck', 'This board is read only')
			} else {
				return t('deck', 'Drop your files to upload')
			}
		},
	},
	watch: {
		cardId: {
			immediate: true,
			handler() {
				this.$store.dispatch('fetchAttachments', this.cardId)
			},
		},
	},
	methods: {
		handleUploadFile(event) {
			const files = event.target.files ?? []
			for (const file of files) {
				this.onLocalAttachmentSelected(file, 'file')
			}
			event.target.value = ''
		},
		uploadNewFile() {
			this.$refs.filesAttachment.click()
		},
		shareFromFiles() {
			picker.pick()
				.then(async (path) => {
					console.debug(`path ${path} selected for sharing`)
					if (!path.startsWith('/')) {
						throw new Error(t('files', 'Invalid path selected'))
					}

					axios.post(generateOcsUrl('apps/files_sharing/api/v1/shares'), {
						path,
						shareType: 12,
						shareWith: '' + this.cardId,
					}).then(() => {
						this.$store.dispatch('fetchAttachments', this.cardId)
					})
				})
		},
		unshareAttachment(attachment) {
			this.$store.dispatch('unshareAttachment', attachment)
		},
		clickAddNewAttachmment() {
			this.$refs.localAttachments.click()
		},
		showViewer(attachment) {
			if (attachment.extendedData.fileid && window.OCA.Viewer.availableHandlers.map(handler => handler.mimes).flat().includes(attachment.extendedData.mimetype)) {
				window.OCA.Viewer.open({ path: attachment.extendedData.path })
				return
			}

			if (attachment.extendedData.fileid) {
				window.location = generateUrl('/f/' + attachment.extendedData.fileid)
				return
			}

			window.location = generateUrl(`/apps/deck/cards/${attachment.cardId}/attachment/${attachment.id}`)
		},
	},
}
</script>

<style lang="scss" scoped>

	.button-group {
		display: flex;

		.icon-upload, .icon-folder {
			padding-left: 44px;
			background-position: 16px center;
			flex-grow: 1;
			height: 44px;
			margin-bottom: 12px;
			text-align: left;
		}
	}

	.attachment-list {
		&.selector {
			padding: 10px;
			position: absolute;
			width: 30%;
			max-width: 500px;
			min-width: 200px;
			max-height: 50%;
			top: 50%;
			left: 50%;
			transform: translate(-50%, -50%);
			background-color: #eee;
			z-index: 2;
			border-radius: 3px;
			box-shadow: 0 0 3px darkgray;
			overflow: scroll;
		}
		h3.attachment-selector {
			margin: 0 0 10px;
			padding: 0;
			.icon-close {
				display: inline-block;
				float: right;
			}
		}

		li.attachment {
			display: flex;
			padding: 3px;
			min-height: 44px;

			&.deleted {
				opacity: .5;
			}

			.fileicon {
				display: inline-block;
				min-width: 32px;
				width: 32px;
				height: 32px;
				background-size: contain;
			}
			.details {
				flex-grow: 1;
				flex-shrink: 1;
				min-width: 0;
				flex-basis: 50%;
				line-height: 110%;
				padding: 2px;
			}
			.filename {
				width: 70%;
				display: flex;
				.basename {
					white-space: nowrap;
					overflow: hidden;
					text-overflow: ellipsis;
					padding-bottom: 2px;
				}
				.extension {
					opacity: 0.7;
				}
			}
			.filesize, .filedate {
				font-size: 90%;
				color: darkgray;
			}
			.app-popover-menu-utils {
				position: relative;
				right: -10px;
				button {
					height: 32px;
					width: 42px;
				}
			}
			button.icon-history {
				width: 44px;
			}
			progress {
				margin-top: 3px;
			}
		}
	}

</style>

<!--
  - @copyright Copyright (c) 2023 Julius Härtl <jus@bitgrid.net>
  -
  - @author Julius Härtl <jus@bitgrid.net>
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
  - MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
  - GNU Affero General Public License for more details.
  -
  - You should have received a copy of the GNU Affero General Public License
  - along with this program. If not, see <http://www.gnu.org/licenses/>.
  -->

<template>
	<div v-if="filePath === null" class="office-target-picker">
		<div ref="picker" class="reference-file-picker" />
	</div>
	<div v-else class="office-target-picker">
		<h2>{{ t('richdocuments', 'Link to office document section') }}</h2>
		<NcLoadingIcon v-if="!sections || !fileId" :size="44" />
		<div v-else>
			<NcEmptyContent v-if="sections.length === 0" :title="t('richdocuments', 'Could not find any section in the document')">
				<template #icon>
					<TableOfContentsIcon />
				</template>
			</NcEmptyContent>
			<template v-else>
				<div v-for="section in sections" :key="section.label">
					<h3>{{ section.label }}</h3>
					<ul>
						<NcListItem v-for="entry in section.entries"
							:key="entry.id"
							:title="entry.name"
							:class="{ 'list-item__wrapper--active': entry.id === target }"
							@click="setTarget(entry)">
							<template v-if="entry.preview" #icon>
								<img :src="entry.preview">
							</template>
						</NcListItem>
					</ul>
				</div>
			</template>
			<div v-if="sections.length !== 0" class="office-target-picker__buttons">
				<NcButton type="primary" :disabled="!target" @click="submit()">
					{{ t('richdocuments', 'Link to office document section') }}
				</NcButton>
			</div>
		</div>
	</div>
</template>

<script>
import { FilePickerType } from '@nextcloud/dialogs'
import { generateUrl, generateOcsUrl } from '@nextcloud/router'
import axios from '@nextcloud/axios'
import { NcButton, NcEmptyContent, NcListItem, NcLoadingIcon } from '@nextcloud/vue'
import TableOfContentsIcon from 'vue-material-design-icons/TableOfContents.vue'

export default {
	name: 'DocumentTargetPicker',
	components: {
		NcButton,
		NcEmptyContent,
		NcListItem,
		NcLoadingIcon,
		TableOfContentsIcon,
	},
	props: {
		providerId: {
			type: String,
			required: true,
		},
		accessible: {
			type: Boolean,
			default: false,
		},
	},
	data() {
		return {
			fileId: null,
			filePath: null,
			target: null,
			sections: null,
		}
	},
	mounted() {
		this.openFilePicker()
		window.addEventListener('click', this.onWindowClick)
	},
	beforeDestroy() {
		window.removeEventListener('click', this.onWindowClick)
	},
	methods: {
		onWindowClick(e) {
			if (e.target.tagName === 'A' && e.target.classList.contains('oc-dialog-close')) {
				this.$emit('cancel')
			}
		},
		async openFilePicker() {
			const self = this
			OC.dialogs.filepicker(
				t('files', 'Select file or folder to link to'),
				function(file) {
					const client = OC.Files.getClient()
					client.getFileInfo(file).then((_status, fileInfo) => {
						self.fileId = fileInfo.id
					})
					self.filePath = file
					self.fetchReferences()
				},
				false, // multiselect
				OC.getCapabilities().richdocuments.mimetypes, // mime filter
				false, // modal
				FilePickerType.Choose, // type
				'',
				{
					target: this.$refs.picker,
				},
			)
		},
		setTarget(entry) {
			this.target = entry.id
		},
		submit() {
			const fileLink = window.location.protocol + '//' + window.location.host
				+ generateUrl('/apps/richdocuments/editonline/{fileId}/{target}', { fileId: this.fileId, target: this.target })
			this.$emit('submit', fileLink)
		},
		async fetchReferences() {
			const response = await axios.get(generateOcsUrl('/apps/richdocuments/api/v1/targets'), {
				params: {
					path: this.filePath,
				},
			})
			this.sections = response.data.ocs.data
		},
	},
}
</script>

<style scoped lang="scss">
.reference-file-picker {
	flex-grow: 1;

	&:deep(.oc-dialog) {
		transform: none !important;
		box-shadow: none !important;
		flex-grow: 1 !important;
		position: static !important;
		width: 100% !important;
		height: auto !important;
		padding: 0 !important;
		max-width: initial;

		.oc-dialog-close {
			display: none;
		}

		.oc-dialog-buttonrow.onebutton.aside {
			position: absolute;
			padding: 12px 32px;
		}

		.oc-dialog-content {
			max-width: 100% !important;
		}
	}
}

.office-target-picker {
	margin: calc(var(--default-grid-baseline) * 4);
	flex-grow: 1;
	height: 80vh;

	&__buttons {
		position: sticky;
		bottom: 12px;
		display: flex;
		align-items: flex-end;
		flex-direction: column;
	}
}

h3 {
	font-weight: bold;
	color: var(--color-primary-element);
	font-size: var(--default-font-size);
	line-height: 44px;
	white-space: nowrap;
	overflow: hidden;
	text-overflow: ellipsis;
	opacity: .7;
	box-shadow: none !important;
	flex-shrink: 0;
}

img {
	max-width: 100px;
	margin-left: 20px;
}
</style>

<template>
	<span :class="{error: sharePermissionEqual(0)}">
		<!-- file -->
		<ActionCheckbox v-if="!isFolder"
			:checked="shareHasPermissions(atomicPermissions.UPDATE)"
			:disabled="saving"
			@update:checked="toggleSharePermissions(atomicPermissions.UPDATE)">
			{{ t('files_sharing', 'Allow editing') }}
		</ActionCheckbox>

		<!-- folder -->
		<template v-if="isFolder && fileHasCreatePermission && config.isPublicUploadEnabled">
			<template v-if="!showCustomPermissionsForm">
				<ActionRadio :checked="sharePermissionEqual(bundledPermissions.READ_ONLY)"
					:value="bundledPermissions.READ_ONLY"
					:name="randomFormName"
					:disabled="saving"
					@change="setSharePermissions(bundledPermissions.READ_ONLY)">
					{{ t('files_sharing', 'Read only') }}
				</ActionRadio>

				<ActionRadio :checked="sharePermissionEqual(bundledPermissions.UPLOAD_AND_EDIT)"
					:value="bundledPermissions.UPLOAD_AND_EDIT"
					:disabled="saving"
					:name="randomFormName"
					@change="setSharePermissions(bundledPermissions.UPLOAD_AND_EDIT)">
					{{ t('files_sharing', 'Allow upload and editing') }}
				</ActionRadio>
				<ActionRadio :checked="sharePermissionEqual(bundledPermissions.FILE_DROP)"
					:value="bundledPermissions.FILE_DROP"
					:disabled="saving"
					:name="randomFormName"
					class="sharing-entry__action--public-upload"
					@change="setSharePermissions(bundledPermissions.FILE_DROP)">
					{{ t('files_sharing', 'File drop (upload only)') }}
				</ActionRadio>

				<!-- custom permissions button -->
				<ActionButton :title="t('files_sharing', 'Custom permissions')"
					@click="showCustomPermissionsForm = true">
					<template #icon>
						<Tune />
					</template>
					{{ sharePermissionsIsBundle ? "" : sharePermissionsSummary }}
				</ActionButton>
			</template>

			<!-- custom permissions -->
			<template v-else>
				<ActionCheckbox :checked="shareHasPermissions(atomicPermissions.CREATE)"
					:disabled="saving"
					@update:checked="toggleSharePermissions(atomicPermissions.CREATE)">
					{{ t('files_sharing', 'Create') }}
				</ActionCheckbox>
				<ActionCheckbox :checked="shareHasPermissions(atomicPermissions.READ)"
					:disabled="saving"
					@update:checked="toggleSharePermissions(atomicPermissions.READ)">
					{{ t('files_sharing', 'Read') }}
				</ActionCheckbox>
				<ActionCheckbox :checked="shareHasPermissions(atomicPermissions.UPDATE)"
					:disabled="saving"
					@update:checked="toggleSharePermissions(atomicPermissions.UPDATE)">
					{{ t('files_sharing', 'Update') }}
				</ActionCheckbox>
				<ActionCheckbox :checked="shareHasPermissions(atomicPermissions.DELETE)"
					:disabled="saving"
					@update:checked="toggleSharePermissions(atomicPermissions.DELETE)">
					{{ t('files_sharing', 'Delete') }}
				</ActionCheckbox>

				<ActionButton @click="showCustomPermissionsForm = false">
					<template #icon>
						<ChevronLeft />
					</template>
					{{ t('files_sharing', 'Bundle permissions') }}
				</ActionButton>
			</template>
		</template>
	</span>
</template>

<script>
import ActionButton from '@nextcloud/vue/dist/Components/ActionButton'
import ActionRadio from '@nextcloud/vue/dist/Components/ActionRadio'
import ActionCheckbox from '@nextcloud/vue/dist/Components/ActionCheckbox'

import SharesMixin from '../mixins/SharesMixin'

import Tune from 'vue-material-design-icons/Tune.vue'
import ChevronLeft from 'vue-material-design-icons/ChevronLeft.vue'

export default {
	name: 'SharePermissionsEditor',

	components: {
		ActionButton,
		ActionCheckbox,
		ActionRadio,
		Tune,
		ChevronLeft,
	},

	mixins: [SharesMixin],

	data() {
		return {
			randomFormName: Math.random().toString(27).substr(2),

			showCustomPermissionsForm: false,

			atomicPermissions: {
				READ: OC.PERMISSION_READ,
				UPDATE: OC.PERMISSION_UPDATE,
				CREATE: OC.PERMISSION_CREATE,
				DELETE: OC.PERMISSION_DELETE,
			},
			bundledPermissions: {
				READ_ONLY: OC.PERMISSION_READ,
				EDIT_ONLY: OC.PERMISSION_UPDATE | OC.PERMISSION_CREATE | OC.PERMISSION_READ | OC.PERMISSION_DELETE,
				UPLOAD_AND_EDIT: OC.PERMISSION_UPDATE | OC.PERMISSION_CREATE | OC.PERMISSION_READ | OC.PERMISSION_DELETE,
				FILE_DROP: OC.PERMISSION_CREATE,
			},
		}
	},

	computed: {
		/**
		 * Return the summary of custom checked permissions.
		 *
		 * @return {string}
		 */
		sharePermissionsSummary() {
			return Object.values(this.atomicPermissions)
				.filter(permission => this.shareHasPermissions(permission))
				.map(permission => {
					switch (permission) {
					case this.atomicPermissions.CREATE:
						return this.t('files_sharing', 'Create')
					case this.atomicPermissions.READ:
						return this.t('files_sharing', 'Read')
					case this.atomicPermissions.UPDATE:
						return this.t('files_sharing', 'Update')
					case this.atomicPermissions.DELETE:
						return this.t('files_sharing', 'Delete')
					default:
						return ''
					}
				})
				.join(', ')
		},

		/**
		 * Return wether the share's permission is a bundle
		 *
		 * @return {boolean}
		 */
		sharePermissionsIsBundle() {
			return Object.values(this.bundledPermissions)
				.map(bundle => this.sharePermissionEqual(bundle))
				.filter(isBundle => isBundle)
				.length > 0
		},

		/**
		 * Is the current share a folder ?
		 * TODO: move to a proper FileInfo model?
		 *
		 * @return {boolean}
		 */
		isFolder() {
			return this.fileInfo.type === 'dir'
		},

		/**
		 * Does the current file/folder have create permissions
		 * TODO: move to a proper FileInfo model?
		 *
		 * @return {boolean}
		 */
		fileHasCreatePermission() {
			return !!(this.fileInfo.permissions & OC.PERMISSION_CREATE)
		},
	},

	mounted() {
		// Show the Custom Permissions view on open if the permissions set is not a bundle.
		this.showCustomPermissionsForm = !this.sharePermissionsIsBundle
	},

	methods: {
		/**
		 * Return wether the share has the exact given permissions.
		 *
		 * @param {number} permissions - the permissions to check.
		 *
		 * @return {boolean}
		 */
		sharePermissionEqual(permissions) {
			// We use the share's permission without PERMISSION_SHARE as it is not relevant here.
			return (this.share.permissions & ~OC.PERMISSION_SHARE) === permissions
		},

		/**
		 * Return wether the share has the given permissions.
		 *
		 * @param {number} permissions - the permissions to check.
		 *
		 * @return {boolean}
		 */
		shareHasPermissions(permissions) {
			return this.share.permissions !== 0 && (this.share.permissions & permissions) === permissions
		},

		/**
		 * Add some permissions to an initial set of permissions.
		 *
		 * @param {number} initialPermission - the initial permissions.
		 * @param {number} permissionsToAdd - the permissions to add.
		 *
		 * @return {number}
		 */
		addPermissions(initialPermission, permissionsToAdd) {
			return initialPermission | permissionsToAdd
		},

		/**
		 * Remove some permissions from an initial set of permissions.
		 *
		 * @param {number} initialPermission - the initial permissions.
		 * @param {number} permissionsToRemove - the permissions to remove.
		 *
		 * @return {number}
		 */
		removePermissions(initialPermission, permissionsToRemove) {
			return initialPermission & ~permissionsToRemove
		},

		/**
		 * Set the share permissions to the given permissions.
		 *
		 * @param {number} permissions - the permissions to set.
		 *
		 * @return {void}
		 */
		setSharePermissions(permissions) {
			this.share.permissions = parseInt(permissions, 10)
			this.queueUpdate('permissions')
		},

		/**
		 * Toggle a given permission.
		 *
		 * @param {number} permissions - the permissions to toggle.
		 *
		 * @return {void}
		 */
		toggleSharePermissions(permissions) {
			if (this.shareHasPermissions(permissions)) {
				this.share.permissions = this.removePermissions(this.share.permissions, permissions)
			} else {
				this.share.permissions = this.addPermissions(this.share.permissions, permissions)
			}

			if (this.share.permissions === 0) {
				// Don't proceed with sending the request if permissions equal 0 as this is not authorized.
				return
			}

			this.queueUpdate('permissions')
		},
	},
}
</script>
<style lang="scss" scoped>
.error {
	::v-deep .action-checkbox__label:before {
		border: 1px solid var(--color-error);
	}
}
</style>

<script setup>
import { message } from 'ant-design-vue';
</script>
<template>
	<div class="home-view-body-container">
		<div class="header-container">
			<div class="header-buttons-container">
				<a-upload v-model:file-list="filesToBeUploaded" :customRequest="uploadFile()">
					<a-button size="medium">
						<i class="fa-solid fa-cloud-arrow-up" style="margin-right: 7px;"></i>
						<upload-outlined></upload-outlined>
						Upload
					</a-button>
				</a-upload>
				<a-button style="margin-left: 10px;" @click="newFolderNameField = !newFolderNameField"><i
						class="fa-solid fa-folder-plus" style="margin-right: 7px;"></i>New Folder</a-button>
				<a-input-group compact style="width: 400px; margin-left: 40px" v-if="newFolderNameField">
					<a-input v-model:value="newFolderName" style="width: calc(100% - 200px)" placeholder="Folder Name" />
					<a-button type="primary" @click="createNew('folder')" :loading="newFolderLoading"><i class="fa-solid fa-check"></i></a-button>
				</a-input-group>
			</div>
		</div>
		<div class="content-container">
			<a-breadcrumb id="breadcrumb-nav" ref="breadcrumbNav">
				<a-breadcrumb-item><a @click="getFolderData(homeFolderId, 'home')"><i class="fa-solid fa-house"></i></a></a-breadcrumb-item>
				<a-breadcrumb-item v-for="dir in nextFolderDirectories"><a @click="getFolderData(dir.id, dir.key)">{{ dir.folderName }}</a></a-breadcrumb-item>
				<!-- <a-breadcrumb-item><a href="">Application Center</a></a-breadcrumb-item>
				<a-breadcrumb-item><a href="">Application List</a></a-breadcrumb-item>
				<a-breadcrumb-item>An Application</a-breadcrumb-item> -->
			</a-breadcrumb>
			<!-- <a-space align="center" style="margin-bottom: 16px">
				CheckStrictly: -->
			<!-- <a-switch v-model:checked="rowSelection.checkStrictly"></a-switch> -->
			<!-- </a-space> -->

			<a-table :columns="columns" :data-source="fileAndFolders" :row-selection="rowSelection">
				<a-empty
					image="https://gw.alipayobjects.com/mdn/miniapp_social/afts/img/A*pevERLJC9v0AAAAAAAAAAABjAQAAAQ/original"
					:image-style="{
					height: '60px',
				}">
					<template #description>
						<span>
							Customize
							<a href="#api">Description</a>
						</span>
					</template>
					<a-button type="primary">Create Now</a-button>
				</a-empty>
				<template #bodyCell="{ column, record }">
					<template v-if="column.key === 'name'">
						<a style="color:#2386F2; cursor: pointer;" @click="navigateToFolder(record)">
							<i class="fa-solid fa-folder" style="margin-right: 7px;"
								v-if="record.type === 'folder'"></i>
							<i class="fa-regular fa-file-lines" v-else style="margin-right: 7px;"></i>
							{{ record.name }}</a>
					</template>
				</template>
			</a-table>
		</div>
	</div>
</template>

<script>
// @ is an alias to /src

export default {
	name: 'HomeView',
	data() {
		return {
			newFolderNameField: false,
			newFolderName: "",
			CONNECTION_NAME: 'crmworkdrive',
			FOLDER_ID: 'zx5fge9ab8a2303ab47888c959942c3a97e34',
			pageData: {},
			folderWithModuleName: {},
			currentCrmModule: {},
			newFolderLoading: false,
			filesToBeUploaded: [],
			openedFolder: {},
			homeFolderId: '',
			nextFolderDirectories: [],
			columns: [
				{
					title: 'Name',
					dataIndex: 'name',
					key: 'name',
				},
				{
					title: 'Size',
					dataIndex: 'size',
					key: 'size',
				},
				{
					title: 'Created By',
					dataIndex: 'createdBy',
					key: 'createdBy',
				},
				{
					title: 'Modified By',
					dataIndex: 'modifiedBy',
					key: 'modifiedBy'
				},
				{
					title: 'Created Time',
					dataIndex: 'createdTime',
					key: 'createdTime',
				},
			],
			fileAndFolders: [],
			rowSelection: {
				checkStrictly: false,
				onChange: (selectedRowKeys, selectedRows) => {
					console.log(`selectedRowKeys: ${selectedRowKeys}`, 'selectedRows: ', selectedRows);
				},
				onSelect: (record, selected, selectedRows) => {
					console.log(record, selected, selectedRows);
				},
				onSelectAll: (selected, selectedRows, changeRows) => {
					console.log(selected, selectedRows, changeRows);
				},
			}

		}
	},

	async created() {
		console.log("Hello");
		await ZOHO.embeddedApp.on("PageLoad", async data => {
			this.pageData = data;
			console.log("Page Data - ", this.pageData);
			console.log("Get Entity - ", this.pageData.Entity);
			const getModuleRecord = await ZOHO.CRM.API.getRecord({
				Entity: this.pageData.Entity, RecordID: this.pageData.EntityId
			});
			console.log("Have Workdrive Folder - ", getModuleRecord.data[0].Have_Workdrive_Folder);
			if(getModuleRecord.data[0].Have_Workdrive_Folder) {
				console.log(getModuleRecord.data[0].Workdrive_Folder_ID);
				const current_folder_id = getModuleRecord.data[0].Workdrive_Folder_ID;
				this.openedFolder = getModuleRecord.data[0];
				this.homeFolderId = current_folder_id;
				await this.getFolderData(current_folder_id);
			}
		});


		try {
			await ZOHO.embeddedApp.init();

		} catch (err) {
			console.log('Failed to initialize SDK');
		}

		await ZOHO.embeddedApp.on("PageLoad", function (data) {
			console.log("Pageload data - ");
		});

		const workdrive_folder_names = [];
		const get_req_folder_url = 'https://www.zohoapis.in/workdrive/api/v1/files/{folder_id}/files?page%5Blimit%5D={limit}&page%5Boffset%5D={offset}';

		const get_folders = await this.getWorkDriveConnection(get_req_folder_url, 0, 50);
		get_folders.details.statusMessage.data.forEach(folder => {
			workdrive_folder_names.push(folder.attributes.name);
			if (folder.attributes.name === this.pageData.Entity) {
				this.folderWithModuleName = folder;
				console.log("Folder with Module - ", this.folderWithModuleName);
			}
		})

		const get_folders_2 = await this.getWorkDriveConnection(get_req_folder_url, 50, 50);
		get_folders_2.details.statusMessage.data.forEach(folder => {
			workdrive_folder_names.push(folder.attributes.name);
		})

		console.log(workdrive_folder_names);

		const get_all_modules = await ZOHO.CRM.META.getModules();
		get_all_modules.modules.forEach(async module => {
			// console.log("Module - ", module);
			// console.log("Module Length - ", module.length);
			if (module.actual_plural_label === this.pageData.Entity) {
				this.currentCrmModule = module;
				console.log("Current CRM Module - ", this.currentCrmModule);
			}
			if (!workdrive_folder_names.includes(module.actual_plural_label)) {
				console.log("Not Included ---------------", module.actual_plural_label);
				await this.postFolderOnWorkDrive(module.actual_plural_label, this.FOLDER_ID);
			} else {
				console.log("Included...........");
			}
		});
	},

	methods: {

		async getWorkDriveConnection(getUrl, offset, limit) {
			return new Promise(async (resolve, reject) => {
				const folder_id = this.FOLDER_ID;

				let req_config = {
					method: "GET",
					headers: {
						'Accept': 'application/vnd.api+json'
					},
					url: getUrl.replace('{folder_id}', folder_id).replace('{offset}', offset).replace('{limit}', limit),
				};

				try {
					const folder_resp = await ZOHO.CRM.CONNECTION.invoke(this.CONNECTION_NAME, req_config);
					resolve(folder_resp);
				} catch (e) {
					reject(e);
				}
			})
		},

		async postFolderOnWorkDrive(folder_name, folder_id) {
			const post_folder_url = 'https://www.zohoapis.in/workdrive/api/v1/files';

			const post_folder_config = {
				method: "POST",
				headers: {
					'Accept': 'application/vnd.api+json'
				},
				parameters: {
					data: {
						attributes: {
							name: folder_name,
							parent_id: folder_id
						},
						type: 'files'
					}
				},
				url: post_folder_url
			};

			try {
				const folder_added_resp = await ZOHO.CRM.CONNECTION.invoke(this.CONNECTION_NAME, post_folder_config);
				console.log(`${folder_name} - Folder Added Successfully`);
				return folder_added_resp;
			} catch (error) {
				console.log('Error Adding Folder');
			}
		},

		async createNew(toCreate) {
			this.newFolderLoading = true;
			const modules_folder_id = this.folderWithModuleName.id;
			console.log(modules_folder_id);
			const getModuleRecord = await ZOHO.CRM.API.getRecord({
				Entity: this.pageData.Entity, RecordID: this.pageData.EntityId
			});
			console.log("Have Workdrive Folder - ", getModuleRecord.data[0].Have_Workdrive_Folder);
			let module_records_folder_id = "";
			let apiData = {};

			if(!getModuleRecord.data[0].Have_Workdrive_Folder) {
				const added_record_folder = await this.postFolderOnWorkDrive(getModuleRecord.data[0].Name, modules_folder_id);
				console.log("Added Record Folder - ", added_record_folder.details.statusMessage.data.id);
				module_records_folder_id = added_record_folder.details.statusMessage.data.id;
				this.openedFolder = added_record_folder.details.statusMessage.data;
				var update_config = {
					Entity: this.pageData.Entity,
					APIData: {
						"id": this.pageData.EntityId,
						"Workdrive_Folder_ID": module_records_folder_id,
						"Have_Workdrive_Folder": true
					}
				};
				await ZOHO.CRM.API.updateRecord(update_config);
			} else {
				// module_records_folder_id = getModuleRecord.data[0].Workdrive_Folder_ID;
				module_records_folder_id = this.openedFolder.id;
			}
			console.log("Module Record ID - ", module_records_folder_id);

			if(toCreate === 'folder') {
				const add_inputed_folder = await this.postFolderOnWorkDrive(this.newFolderName, module_records_folder_id);
				const get_added_folder_details = add_inputed_folder.details.statusMessage.data;
				console.log(get_added_folder_details);
				const new_folder_data = {
					key: this.fileAndFolders.length + 1,
					name: get_added_folder_details.attributes.name,
					size: '120 MB',
					type: 'folder',
					createdBy: get_added_folder_details.attributes.created_by,
					modifiedBy: get_added_folder_details.attributes.modified_by,
					createdTime: get_added_folder_details.attributes.created_time,
					moreDetails: get_added_folder_details
				};
				this.fileAndFolders.unshift(new_folder_data);
				console.log("New Folder Name - ", this.newFolderName);
				this.newFolderName = "";
				console.log(getModuleRecord.data[0]);
				this.newFolderLoading = false;
				this.newFolderNameField = false;
				message['success']({
					top: '30px',
					duration: 2,
					content: 'Folder Created Successfully',
					rtl: true,
					style: {
						fontSize: '15px',
						marginTop: '3px'
					},
				})
			}

			if(toCreate === 'file') {
				// return ({ file, onSuccess, onError }) => {
					// return new Promise((resolve, reject) => {
						// console.log('uploadedFile - ', file);
						// onSuccess();
						// return resolve();
					// })
				// }
			}
		},

		async getFolderData(current_folder_id, directoryKey) {
			const get_folder_config = {
				method: 'GET',
				headers: {
					'Accept': 'application/vnd.api+json'
				},
				url: `https://www.zohoapis.in/workdrive/api/v1/files/${current_folder_id}/files`
			};
			const get_current_folder_resp = await ZOHO.CRM.CONNECTION.invoke(this.CONNECTION_NAME, get_folder_config);
			console.log('Get Folder - ', get_current_folder_resp.details.statusMessage.data);
			this.fileAndFolders = [];
			get_current_folder_resp.details.statusMessage.data.forEach((folder, index) => {
				let folderData = {
					key: index + 1,
					name: folder.attributes.name,
					size: '120 MB',
					type: folder.attributes.type,
					createdBy: folder.attributes.created_by,
					modifiedBy: folder.attributes.modified_by,
					createdTime: folder.attributes.created_time,
					moreDetails: folder
				}
				this.fileAndFolders.push(folderData);
			})
			if(directoryKey) {
				console.log("Directory Key");
				if(directoryKey === 'home') {
					this.nextFolderDirectories = [];
				} else {
					this.nextFolderDirectories.forEach((dir, index) => {
						if(dir.key > directoryKey) {
							this.nextFolderDirectories.splice(index);
							// break;
						}
					})
				}
			}
		},

		async navigateToFolder(getDetails) {
			if(getDetails.moreDetails.attributes.type === 'folder') {
				console.log(getDetails.moreDetails.id);
				this.openedFolder = getDetails.moreDetails;
				await this.getFolderData(this.openedFolder.id);
				this.nextFolderDirectories.push({ key: this.nextFolderDirectories.length + 1, id: this.openedFolder.id, folderName: this.openedFolder.attributes.name })
			}
		},

		uploadFile() {
			return ({ file, onSuccess, onError }) => {
				console.log("File - ", file);
				setTimeout(async () => {
					// const post_file_config = {
					// 	method: "POST",
					// 	headers: {
					// 		'Accept': 'application/vnd.api+json',
					// 		'x-filename': 'testingImage.jpg',
					// 		'x-parent_id': '4xsh2923e1131bb2e4b4c9fb755761779cb29',
					// 		'x-streammode': '1'
					// 	},
					// 	parameters: file,
					// 	contentType : "application/octet-stream",
					// 	url: 'https://upload.zoho.in/workdrive-api/v1/stream/upload'
					// };

					const fileFormData = new FormData();
					fileFormData.append('filename', 'imageTesting.jpg');
					fileFormData.append('parent_id', '4xsh2923e1131bb2e4b4c9fb755761779cb29');
					fileFormData.append('override-name-exist', 'true');
					fileFormData.append('content', file);

					const post_file_config = {
						method: "POST",
						headers: {
							'Accept': 'application/vnd.api+json'
						},
						parameters: {
							data: {
								attributes: {
									fileFormData
								},
							}
						},
						contentType: "multipart/form-data",
						url: 'https://www.zohoapis.in/workdrive/api/v1/upload'
					};
					const fileUploadResp = await ZOHO.CRM.CONNECTION.invoke(this.CONNECTION_NAME, post_file_config);
					console.log("File Upload Response - ", fileUploadResp);
					onSuccess();
				}, 5000);
			}
		}
	}
}
</script>

<style>
.home-view-body-container {
	padding: 20px;
}

.ant-table-wrapper .ant-table-thead>tr>th,
.ant-table-wrapper .ant-table-thead>tr>td {
	position: relative;
	color: rgb(255, 255, 255);
	text-align: start;
	background-color: #3d556d !important;
	font-weight: 700 !important;
	border-bottom: 1px solid #3d556d !important;
	border-inline-end: 1px solid #3d556d;
	transition: background 0.2s ease;

}

.header-buttons-container {
	display: flex;
	padding-bottom: 20px;
}

#breadcrumb-nav {
	padding-left: 3px;
	padding-bottom: 6px;
}

#breadcrumb-nav a {
	/* color: #2386F2; */
	cursor: pointer;
}
</style>
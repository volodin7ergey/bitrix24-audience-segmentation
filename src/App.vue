<script setup lang="ts">
import { reactive, ref, computed, onMounted } from 'vue'

const headers = [
	{ title: 'Фамилия', key: 'lastName' },
	{ title: 'Имя', key: 'name' },
	{ title: 'Отчество', key: 'secondName' },
	{ title: 'Email', key: 'email' },
	{ title: 'Телефон', key: 'phone' },
	{ title: 'Должность', key: 'post' },
	{ title: 'Компания', key: 'company' },

	{ title: 'Название сделки', key: 'dealTitle' },
	{ title: 'Тип сделки', key: 'dealType' },
	{ title: 'Стадия сделки', key: 'dealStage' },
	{ title: 'Направление сделки', key: 'dealCategory' },

	{ title: 'Юр. лицо', key: 'legalEntity' },
	{ title: 'Город доставки', key: 'cityDelivery' },
	{ title: 'Выставка', key: 'exhibition' },
	{ title: 'Оборудование', key: 'equipment' },
	{ title: 'Согласие', key: 'agreement' },
]

const formatCell = (value: any, emptyLabel = '—') => {
	if (value === null || value === undefined) return emptyLabel

	if (Array.isArray(value)) {
		if (value.length === 0) return emptyLabel
		return value.join(', ')
	}

	if (typeof value === 'string' && value.trim() === '') return emptyLabel

	return value
}

const isSendModalOpen = ref(false)

const sendMode = ref<'dry' | 'real'>('dry')

const sendProgress = ref(0)
const sendTotal = ref(0)

const sendLog = ref<string[]>([])

const isSending = ref(false)

const sendStep = ref<1 | 2>(1)
const dryRunDone = ref(false)

const openSendModal = () => {
	sendMode.value = 'dry'
	sendProgress.value = 0
	sendLog.value = []
	isSendModalOpen.value = true
}

const api = <T,>(
	method: string,
	params: Record<string, any> = {},
): Promise<T> =>
	new Promise((resolve, reject) => {
		BX24.callMethod(method, params, (result: any) => {
			if (result.error()) reject(result.error())
			else resolve(result.data())
		})
	})

const isAppReady = ref(false)

onMounted(() => {
	setTimeout(() => {
		isAppReady.value = true
	}, 1000)
})

const getUserCategories = async (): Promise<IUserDealCategoryDto[]> =>
	await api('crm.dealcategory.list')


const getDefaultCategory = async (): Promise<IDefaultDealCategoryDto> =>
	await api('crm.dealcategory.default.get')

const getDealCategoriesWithStages = async () => {
	const [userCategories, defaultCategory] = await Promise.all([
		getUserCategories(),
		getDefaultCategory(),
	])

	const all = [...userCategories, defaultCategory]
	const result: any[] = []

	for (const cat of all) {
		const entityId = Number(cat.ID) > 0 ? `DEAL_STAGE_${cat.ID}` : 'DEAL_STAGE'

		const stages = await api('crm.status.list', {
			filter: { ENTITY_ID: entityId },
		})

		result.push({
			id: cat.ID,
			name: cat.NAME,
			//@ts-ignore
			stages: stages.map((s: any) => ({
				STATUS_ID: s.STATUS_ID,
				NAME: s.NAME,
				SEMANTICS: s.SEMANTICS, // 🔥 ВАЖНО
			})),
		})
	}

	return result
}

const CATEGORY_IDS = reactive([0, 2, 4, 6, 8, 10])

const USER_FIELDS = {
	legalEntity: 'UF_CRM_1774202928053',
	cityDelivery: 'UF_CRM_1774203270529',
	exhibition: 'UF_CRM_1774203630109',
	equipment: 'UF_CRM_1774204022371',
}

const AGREEMENT_RULES: Record<number, string[]> = {
	0: ['UC_DHOGIC'], // Бюджетные продукты
	8: [], // Отложенные → всегда false
}

const CATEGORY_TO_LIST: Record<number, number> = {
	0: 3,
}

const normalizeContact = (contact: any) => ({
	name: `${contact.NAME || ''} ${contact.LAST_NAME || ''}`.trim(),
	email: contact.HAS_EMAIL === 'Y' ? contact.EMAIL?.[0]?.VALUE || null : null,
	phone: contact.HAS_PHONE === 'Y' ? contact.PHONE?.[0]?.VALUE || null : null,
})

type NormalizedAddress = {
	raw: string | null
	text: string | null
	lat: number | null
	lon: number | null
	hasCoords: boolean
}

const parseAddressString = (value: string | null): NormalizedAddress => {
	if (!value) {
		return { raw: null, text: null, lat: null, lon: null, hasCoords: false }
	}

	const parts = value.split('|')
	const text = parts[0] || null
	const coords = parts[1] || ''

	let lat: number | null = null
	let lon: number | null = null

	if (coords.includes(';')) {
		const [latStr, lonStr] = coords.split(';')
		lat = latStr ? Number(latStr) : null
		lon = lonStr ? Number(lonStr) : null
	}

	return {
		raw: value,
		text,
		lat,
		lon,
		hasCoords: lat !== null && lon !== null,
	}
}

const computeAgreement = (deal: any, stageMetaMap: Record<string, any>) => {
	const categoryId = Number(deal.CATEGORY_ID)
	const stageId = deal.STAGE_ID

	// 1. Отложенные → всегда false
	if (categoryId === 8) return false


	const stageKey = `${categoryId}:${stageId}`
	const stageMeta = stageMetaMap[stageKey]

	if (stageMeta?.SEMANTICS === 'S') return true

	const allowedStages = AGREEMENT_RULES[categoryId] || []

	return allowedStages.includes(stageId)
}

const cleanObject = (obj: Record<string, any>) => {
	const result: Record<string, any> = {}

	Object.entries(obj).forEach(([key, value]) => {
		if (value === null || value === undefined) return

		if (Array.isArray(value) && value.length === 0) return

		if (typeof value === 'string' && value.trim() === '') return

		result[key] = value
	})

	return result
}

const DEFAULT_LIST_ID = 3

const mapToUnisender = (row: any) => {
	return {
		email: row.email,
		list_id: DEFAULT_LIST_ID,
		fields: cleanObject({
			Name: row.name,
			last_name: row.lastName,
			surname: row.secondName,
			email: row.email,
			telefon: row.phone,
			jobtitle: row.post,
			company: row.company,

			legal_entity: row.legalEntity,
			city: row.cityDelivery,
			exhibition: row.exhibition,
			equipment: Array.isArray(row.equipment)
				? row.equipment.join(', ')
				: row.equipment,

			stage_deal: row.dealStage,
			transation: row.dealType,
			agreement: row.agreement ? 'true' : 'false',
		}),
	}
}

const normalizeAddress = (field: any) => {
	if (!field) return null
	if (Array.isArray(field)) return field.map(parseAddressString)
	return parseAddressString(field)
}

const normalizeEnum = (value: any, fieldMeta: any) => {
	if (!fieldMeta?.items) return null

	const map: Record<string, string> = {}
	fieldMeta.items.forEach((i: any) => {
		map[String(i.ID)] = i.VALUE
	})

	const isMultiple = fieldMeta.isMultiple === true

	if (isMultiple) {
		if (!value) return []
		const arr = Array.isArray(value) ? value : [value]
		return arr.map(v => map[String(v)]).filter(Boolean)
	}

	if (!value) return null
	return map[String(value)] || null
}

// ---------------- API ----------------
const getDealFields = async () => await api('crm.deal.fields')

const getStages = async () => {
	const categories = await api('crm.dealcategory.list')
	const map: Record<string, string> = {}

	for (const cat of categories) {
		const stages = await api('crm.dealcategory.stage.list', {
			CATEGORY_ID: cat.ID,
		})

		stages.forEach((s: any) => {
			map[`${cat.ID}:${s.STATUS_ID}`] = s.NAME
		})
	}

	return map
}

const getAllDeals = () =>
	new Promise<any[]>((resolve, reject) => {
		BX24.callMethod(
			'crm.deal.list',
			{
				filter: { CATEGORY_ID: CATEGORY_IDS },
				select: ['ID'],
			},
			res => {
				if (res.error()) return reject(res.error())

				const total = res.total()
				const limit = 50
				const batches = Math.ceil(total / limit)
				const cmd: any = {}

				for (let i = 0; i < batches; i++) {
					cmd[`d${i}`] = [
						'crm.deal.list',
						{
							select: [
								'ID',
								'TITLE',
								'CATEGORY_ID',
								'STAGE_ID',
								'TYPE_ID',
								'CONTACT_ID',
								...Object.values(USER_FIELDS),
							],
							filter: { CATEGORY_ID: CATEGORY_IDS },
							order: { ID: 'DESC' },
							start: i * limit,
						},
					]
				}

				BX24.callBatch(cmd, batchRes => {
					let deals: any[] = []

					Object.values(batchRes).forEach((r: any) => {
						if (!r.error()) deals = deals.concat(r.data())
					})

					resolve(deals)
				})
			},
		)
	})

const getDealContactsBatch = async (dealIds: number[]) => {
	const map: Record<number, number[]> = {}

	for (let i = 0; i < dealIds.length; i += 50) {
		const chunk = dealIds.slice(i, i + 50)
		const cmd: Record<string, string> = {}

		chunk.forEach(id => {
			cmd[`dc${id}`] = `crm.deal.contact.items.get?id=${id}`
		})

		const res: any = await api('batch', { halt: 0, cmd })

		Object.entries(res.result).forEach(([key, value]: any) => {
			const dealId = Number(key.replace('dc', ''))
			map[dealId] = value.map((v: any) => Number(v.CONTACT_ID))
		})
	}

	return map
}

const getContactsBatch = async (ids: number[]) => {
	const map: Record<number, any> = {}

	for (let i = 0; i < ids.length; i += 50) {
		const chunk = ids.slice(i, i + 50)
		const cmd: Record<string, string> = {}

		chunk.forEach(id => (cmd[`c${id}`] = `crm.contact.get?id=${id}`))

		const res: any = await api('batch', { halt: 0, cmd })

		Object.entries(res.result).forEach(([key, value]: any) => {
			map[Number(key.replace('c', ''))] = value
		})
	}

	return map
}

const getCompaniesBatch = async (ids: number[]) => {
	const map: Record<number, any> = {}

	for (let i = 0; i < ids.length; i += 50) {
		const chunk = ids.slice(i, i + 50)
		const cmd: Record<string, string> = {}

		chunk.forEach(id => (cmd[`cmp${id}`] = `crm.company.get?id=${id}`))

		const res: any = await api('batch', { halt: 0, cmd })

		Object.entries(res.result).forEach(([key, value]: any) => {
			map[Number(key.replace('cmp', ''))] = value
		})
	}

	return map
}

const getDealTypes = async () =>
	await await api('crm.status.list', {
		filter: { ENTITY_ID: 'DEAL_TYPE' },
	})

const dealsData = ref<any[]>([])
const isLoading = ref<boolean>(false)

const usApiKey = ref<string>('')

const isUsLoading = ref(false)

const loadData = async () => {
	isLoading.value = true

	try {
		const [deals, fields, categories, dealTypes] = await Promise.all([
			getAllDeals(),
			getDealFields(),
			getDealCategoriesWithStages(),
			getDealTypes(),
		])

		const typeStatusMap: Record<string, string> = {}

		dealTypes.forEach(s => {
			typeStatusMap[s.STATUS_ID] = s.NAME
		})

		const stageMap: Record<string, string> = {}
		categories.forEach(cat => {
			cat.stages.forEach((stage: any) => {
				stageMap[`${cat.id}:${stage.STATUS_ID}`] = stage.NAME
			})
		})

		const categoryMap: Record<number, string> = {}

		categories.forEach(cat => {
			categoryMap[Number(cat.id)] = cat.name
		})

		const stageMetaMap: Record<string, any> = {}

		categories.forEach(cat => {
			cat.stages.forEach((stage: any) => {
				stageMetaMap[`${cat.id}:${stage.STATUS_ID}`] = stage
			})
		})

		const dealIds = deals.map(d => Number(d.ID))
		const dealContactsMap = await getDealContactsBatch(dealIds)

		const contactToDeals: Record<number, any[]> = {}

		deals.forEach(deal => {
			const contacts = dealContactsMap[deal.ID] || []

			contacts.forEach(cid => {
				if (!contactToDeals[cid]) contactToDeals[cid] = []
				contactToDeals[cid].push(deal)
			})
		})

		const contactIds = Object.keys(contactToDeals).map(Number)

		const contactsMap = await getContactsBatch(contactIds)

		const companyIds = [
			...new Set(
				Object.values(contactsMap)
					.map((c: any) => c.COMPANY_ID)
					.filter(Boolean),
			),
		]

		const companiesMap = await getCompaniesBatch(companyIds)

		const result = contactIds.map(cid => {
			const deals = contactToDeals[cid]

			const bestDeal = deals.sort((a, b) => Number(b.ID) - Number(a.ID))[0]

			const contact = contactsMap[cid]
			if (!contact) return null

			const normalized = normalizeContact(contact)
			if (!normalized.email) return null

			const company = companiesMap[contact.COMPANY_ID]

			const city = normalizeAddress(bestDeal[USER_FIELDS.cityDelivery])

			return {
				contactId: cid,
				name: contact.NAME || null,
				lastName: contact.LAST_NAME || null,
				secondName: contact.SECOND_NAME || null,
				email: normalized.email,
				phone: normalized.phone,
				post: contact.POST || null,
				company: company?.TITLE || null,

				dealTitle: bestDeal.TITLE,
				dealCategoryId: bestDeal.CATEGORY_ID,
				dealType: typeStatusMap[bestDeal.TYPE_ID] || 'Продажа',
				dealStage:
					stageMap[`${bestDeal.CATEGORY_ID}:${bestDeal.STAGE_ID}`] || null,
				dealCategory:
					categoryMap[Number(bestDeal.CATEGORY_ID)] || 'Без названия',
				legalEntity: normalizeEnum(
					bestDeal[USER_FIELDS.legalEntity],
					fields[USER_FIELDS.legalEntity],
				),
				cityDelivery: Array.isArray(city) ? city[0]?.text : city?.text,
				exhibition: normalizeEnum(
					bestDeal[USER_FIELDS.exhibition],
					fields[USER_FIELDS.exhibition],
				),
				equipment: normalizeEnum(
					bestDeal[USER_FIELDS.equipment],
					fields[USER_FIELDS.equipment],
				),
				agreement: computeAgreement(bestDeal, stageMetaMap), // ✅ правильно
			}
		})

		dealsData.value = result.filter(Boolean)

		isLoading.value = false
	} catch (e) {
		console.error(e)
	} finally {
		isLoading.value = false
	}
}

const agreementCount = computed(
	() => dealsData.value.filter(d => d.agreement).length,
)

const noPhoneCount = computed(
	() => dealsData.value.filter(d => !d.phone).length,
)

const noEquipmentCount = computed(
	() =>
		dealsData.value.filter(d => !d.equipment || d.equipment.length === 0)
			.length,
)


const runDry = async () => {
	sendLog.value = []
	sendProgress.value = 0

	const payload = dealsData.value.map(mapToUnisender)

	sendTotal.value = payload.length

	for (let i = 0; i < payload.length; i++) {
		const item = payload[i]

		sendProgress.value++ 

		await sleep(100) 
	}

	dryRunDone.value = true
	sendStep.value = 2
}

const isSuccess = ref(false)
const sleep = (ms: number) => new Promise(res => setTimeout(res, ms))

const sendReal = async () => {
	isSending.value = true
	isSuccess.value = false

	const payload = dealsData.value.map(mapToUnisender)

	sendTotal.value = payload.length
	sendProgress.value = 0

	for (const item of payload) {
		const body = new URLSearchParams()

		body.append('api_key', usApiKey.value)
		body.append('list_ids', String(item.list_id))
		body.append('email', item.email)

		Object.entries(item.fields).forEach(([key, value]) => {
			body.append(`fields[${key}]`, String(value))
		})

		body.append('double_optin', '3')
		body.append('overwrite', '2')

		try {
			await fetch('https://api.unisender.com/ru/api/subscribe', {
				method: 'POST',
				body,
			})
		} catch (e) {
			console.error('SEND ERROR', item.email)
		}

		sendProgress.value++
		await sleep(400)
	}

	isSending.value = false
	isSuccess.value = true

	setTimeout(() => {
		isSendModalOpen.value = false
		sendStep.value = 1
		isSuccess.value = false
	}, 1500)
}
</script>

<template>
	<v-fade-transition>
		<div v-if="!isAppReady" class="app-loader">
			<v-progress-circular indeterminate size="60" width="5" color="primary" />

			<div class="text-h6 mt-4">Загрузка приложения...</div>
		</div>
	</v-fade-transition>
	<div v-if="isAppReady" style="padding: 20px">
		<v-card class="pa-4 mb-4" elevation="2">
			<v-toolbar flat density="comfortable" class="p-2">
				<v-toolbar-title class="text-h6"> Управление данными </v-toolbar-title>

				<v-spacer />
				<v-btn
					color="primary"
					variant="flat"
					:loading="isLoading"
					:disabled="isLoading"
					@click="loadData"
					class="mr-2"
				>
					Загрузить
				</v-btn>

				<v-btn
					color="success"
					variant="flat"
					:disabled="!dealsData.length || isSending"
					@click="openSendModal"
				>
					Отправить в Unisender
				</v-btn>
			</v-toolbar>
		</v-card>

		<v-alert
			v-if="!dealsData.length && !isLoading"
			type="info"
			variant="tonal"
			border="start"
			class="mb-4"
		>
			<div class="text-h6 mb-2">Начало работы</div>

			<div class="mb-2">
				Чтобы отправить данные в Unisender, выполните 3 шага:
			</div>

			<ol class="ml-4">
				<li>Нажмите <b>«Загрузить»</b> — данные подтянутся из Bitrix24</li>
				<li>Проверьте таблицу контактов</li>
				<li>Нажмите <b>«Отправить в US»</b></li>
			</ol>
		</v-alert>

		<v-dialog v-model="isSendModalOpen" max-width="520">
			<v-card class="p-4">
				<v-card-title> Отправка в Unisender </v-card-title>

				<v-chip class="mb-3" color="primary"> Шаг {{ sendStep }} / 2 </v-chip>

				<div v-if="sendStep === 1">
					<div class="text-subtitle-2 mb-2">Проверка данных (Dry run)</div>
				</div>

				<div v-else>
					<div class="text-subtitle-2 mb-2">Подтверждение отправки</div>

					<v-alert type="warning" variant="tonal" class="mb-2">
						Будет отправлено: {{ sendTotal }} контактов
					</v-alert>
				</div>

				<v-alert v-if="isSuccess" type="success" variant="tonal" class="mb-3">
					Отправка успешно завершена
				</v-alert>

				<v-progress-linear
					:model-value="sendProgress"
					:max="sendTotal"
					class="my-3"
				/>

				<div class="text-caption mb-2">
					{{ sendProgress }} / {{ sendTotal }}
				</div>

				<v-card-actions>
					<v-spacer />

					<v-btn @click="isSendModalOpen = false"> Закрыть </v-btn>

					<v-btn
						v-if="sendStep === 1"
						color="primary"
						:loading="isSending"
						@click="runDry"
					>
						Запустить проверку
					</v-btn>


					<v-btn v-else color="red" :loading="isSending" @click="sendReal">
						Подтвердить отправку
					</v-btn>
				</v-card-actions>
			</v-card>
		</v-dialog>
		<v-row class="mb-4" v-if="dealsData.length">
			<v-col cols="12" md="3">
				<v-card class="metric-card primary">
					<div class="metric-top">
						<v-icon size="18">mdi-database</v-icon>
						<span>Всего</span>
					</div>

					<div class="metric-value">
						{{ dealsData.length }}
					</div>
				</v-card>
			</v-col>

			<v-col cols="12" md="3">
				<v-card class="metric-card success">
					<div class="metric-top">
						<v-icon size="18">mdi-check-circle</v-icon>
						<span>С согласием</span>
					</div>

					<div class="metric-value">
						{{ agreementCount }}
					</div>
				</v-card>
			</v-col>

			<v-col cols="12" md="3">
				<v-card class="metric-card warning">
					<div class="metric-top">
						<v-icon size="18">mdi-phone-off</v-icon>
						<span>Без телефона</span>
					</div>

					<div class="metric-value">
						{{ noPhoneCount }}
					</div>
				</v-card>
			</v-col>

			<v-col cols="12" md="3">
				<v-card class="metric-card error">
					<div class="metric-top">
						<v-icon size="18">mdi-cog-off</v-icon>
						<span>Без оборудования</span>
					</div>

					<div class="metric-value">
						{{ noEquipmentCount }}
					</div>
				</v-card>
			</v-col>
		</v-row>
		<v-data-table
			v-if="dealsData.length"
			:headers="headers"
			:items="dealsData"
			item-key="contactId"
			class="airtable-table"
		>
			<template #item="{ item }">
				<tr class="airtable-row">
					<td>{{ item.lastName || '—' }}</td>
					<td>{{ item.name || '—' }}</td>
					<td>{{ item.secondName || '—' }}</td>

					<td>
						<v-chip
							v-if="item.email"
							size="x-small"
							color="primary"
							variant="tonal"
						>
							{{ item.email }}
						</v-chip>
						<span v-else class="empty">—</span>
					</td>

					<td>
						<v-chip v-if="item.phone" size="x-small" color="success">
							{{ item.phone }}
						</v-chip>
						<span v-else class="empty">—</span>
					</td>

					<td>{{ item.post || '—' }}</td>

					<td class="strong">
						{{ item.company || '—' }}
					</td>

					<td>{{ item.dealTitle || '—' }}</td>
					<td>{{ item.dealType || '—' }}</td>
					<td>{{ item.dealStage || '—' }}</td>
					<td>{{ item.dealCategory || '—' }}</td>

					<td>{{ item.legalEntity || '—' }}</td>
					<td>{{ item.cityDelivery || '—' }}</td>
					<td>{{ item.exhibition || '—' }}</td>
					<td>
						<div v-if="Array.isArray(item.equipment) && item.equipment.length">
							<v-chip
								v-for="(eq, i) in item.equipment.slice(0, 2)"
								:key="i"
								size="x-small"
								class="mr-1 mb-1"
								variant="tonal"
								color="blue"
							>
								{{ eq }}
							</v-chip>

							<v-tooltip v-if="item.equipment.length > 2">
								<template #activator="{ props }">
									<v-chip v-bind="props" size="x-small" variant="outlined">
										+{{ item.equipment.length - 2 }}
									</v-chip>
								</template>

								<span>
									{{ item.equipment.join(', ') }}
								</span>
							</v-tooltip>
						</div>

						<span v-else class="empty"> нет оборудования </span>
					</td>

					<td>
						<v-tooltip>
							<template #activator="{ props }">
								<div class="d-flex align-center" v-bind="props">
									<v-icon
										:color="item.agreement ? 'success' : 'grey-lighten-1'"
										size="18"
										class="mr-1"
									>
										{{
											item.agreement
												? 'mdi-check-circle'
												: 'mdi-close-circle-outline'
										}}
									</v-icon>

									<span
										:class="item.agreement ? 'text-success' : 'text-grey'"
										style="font-size: 12px"
									>
										{{ item.agreement ? 'Есть' : 'Нет' }}
									</span>
								</div>
							</template>

							<span>
								{{
									item.agreement
										? 'Контакт дал согласие на коммуникацию'
										: 'Нет согласия на коммуникацию'
								}}
							</span>
						</v-tooltip>
					</td>
				</tr>
			</template>
		</v-data-table>
	</div>
</template>

<style scoped>
.app-loader {
	position: fixed;
	inset: 0;
	background: #f5f7fb;
	display: flex;
	flex-direction: column;
	align-items: center;
	justify-content: center;
	z-index: 9999;
}

.airtable-table {
	border-radius: 12px;
	overflow: hidden;
}

.airtable-row:hover {
	background: #f6f8fb;
	transition: 0.15s ease;
}

.empty {
	color: #b0b7c3;
}

.strong {
	font-weight: 500;
	color: #1f2937;
}

.v-data-table-header {
	position: sticky;
	top: 0;
	background: white;
	z-index: 2;
}

.metric-card {
	padding: 16px;
	border-radius: 14px;
	transition: 0.2s ease;
	cursor: default;
	position: relative;
	overflow: hidden;
}

.metric-card:hover {
	transform: translateY(-2px);
	box-shadow: 0 8px 20px rgba(0, 0, 0, 0.06);
}

.metric-top {
	display: flex;
	align-items: center;
	gap: 6px;
	font-size: 12px;
	color: #6b7280;
}

.metric-value {
	font-size: 28px;
	font-weight: 600;
	margin-top: 8px;
}

.metric-card.primary {
	background: linear-gradient(135deg, #eef4ff, #f7faff);
}

.metric-card.success {
	background: linear-gradient(135deg, #ecfdf5, #f0fdf4);
}

.metric-card.warning {
	background: linear-gradient(135deg, #fffbeb, #fefce8);
}

.metric-card.error {
	background: linear-gradient(135deg, #fef2f2, #fff1f2);
}

.send-log {
	max-height: 180px;
	overflow: auto;
	background: #f8fafc;
	border-radius: 10px;
	padding: 10px;
	font-size: 12px;
	font-family: monospace;
}

.log-line {
	padding: 2px 0;
	color: #334155;
}
</style>
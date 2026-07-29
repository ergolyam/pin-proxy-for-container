<template>
	<div class="panel">
		<div v-if="view === 'containers'" class="menu-panel">
			<h3 class="title">
				<span class="title-text ellipsis">Manage Proxies in Containers</span>
			</h3>
			<hr>
			<div class="scrollable identities-list">
				<div v-if="toggleError" class="list-error" role="alert">
					{{ toggleError }}
				</div>
				<table
					v-if="containerRows.length > 0"
					class="menu menu--containers"
					id="picker-identities-list"
				>
					<tr
						v-for="container in containerRows"
						:key="container.cookieStoreId"
						class="menu-item hover-highlight keyboard-nav"
						tabindex="0"
						@click="openContainer(container.cookieStoreId)"
						@keydown.enter.prevent.self="openContainer(container.cookieStoreId)"
					>
						<td>
							<div class="menu-icon hover-highlight">
								<div
									class="usercontext-icon"
									:data-identity-icon="container.icon"
									:data-identity-color="container.color"
								></div>
							</div>
							<span class="menu-text ellipsis" :title="tooltipText(container.name, 22)">
								{{ container.name }}
							</span>
							<span class="menu-right-float">
								<button
									v-if="container.hasProxy"
									class="proxy-toggle"
									:class="{ 'proxy-toggle--active': container.proxyEnabled }"
									type="button"
									role="switch"
									:aria-checked="container.proxyEnabled"
									:aria-label="proxyToggleLabel(container)"
									:title="proxyToggleTitle(container)"
									:disabled="isProxyTogglePending(container.cookieStoreId)"
									@click.stop="toggleProxy(container)"
								>
									<span class="proxy-toggle-track" aria-hidden="true">
										<span class="proxy-toggle-thumb"></span>
									</span>
									<span class="proxy-toggle-label">
										{{ container.proxyEnabled ? 'On' : 'Off' }}
									</span>
								</button>
								<span v-else class="proxy-state" title="Proxy is not configured">
									—
								</span>
								<span class="menu-arrow">
									<img alt="" src="/img/arrow-icon-right.svg">
								</span>
							</span>
						</td>
					</tr>
				</table>
				<div v-else class="empty-state">
					{{ loadError || 'No Firefox containers found.' }}
				</div>
			</div>
		</div>

		<div v-else class="menu-panel edit-container-proxy">
			<h3 class="title">
				<span class="title-text ellipsis" :title="tooltipText(selectedContainerName, 20)">
					{{ selectedContainerName }}
				</span>
			</h3>
			<button
				class="btn-return arrow-left controller"
				type="button"
				id="close-container-proxy-panel"
				@click="backToContainers"
				aria-label="Back"
			></button>
			<hr>

			<div class="proxy-form scrollable">
				<div class="form-header">
					<div>
						<div class="sub-header">Proxy for this container</div>
						<div v-if="proxyPreview" class="proxy-preview ellipsis" :title="proxyPreview">
							{{ proxyPreview }}
						</div>
					</div>
					<span v-if="hasSelectedAssignment" class="configured-badge">Configured</span>
				</div>

				<div class="form-field">
					<label for="proxy-protocol">Protocol</label>
					<select
						id="proxy-protocol"
						v-model="form.type"
						@change="onProtocolChanged"
					>
						<option
							v-for="protocol in protocols"
							:key="protocol.value"
							:value="protocol.value"
						>
							{{ protocol.label }}
						</option>
					</select>
				</div>

				<div class="address-row">
					<div class="form-field">
						<label for="proxy-host">IP address or host</label>
						<input
							id="proxy-host"
							type="text"
							v-model="form.host"
							placeholder="127.0.0.1"
							maxlength="255"
							autocomplete="off"
							spellcheck="false"
							:aria-invalid="showHostError"
							@input="onFieldInput('host')"
							@blur="saveImmediately"
						>
						<span v-if="showHostError" class="field-error">
							Enter a valid IP address or host.
						</span>
					</div>

					<div class="form-field form-field--port">
						<label for="proxy-port">Port</label>
						<input
							id="proxy-port"
							type="text"
							v-model="form.port"
							:placeholder="portPlaceholder"
							maxlength="5"
							inputmode="numeric"
							autocomplete="off"
							:aria-invalid="showPortError"
							@input="onFieldInput('port')"
							@blur="saveImmediately"
						>
						<span v-if="showPortError" class="field-error">
							Use 1–65535.
						</span>
					</div>
				</div>

				<transition name="auth-fields">
					<div v-if="supportsAuthentication" class="authentication-fields">
						<div class="authentication-heading">
							Authentication <span>Optional</span>
						</div>
						<div class="credentials-row">
							<div class="form-field">
								<label for="proxy-username">Login</label>
								<input
									id="proxy-username"
									type="text"
									v-model="form.username"
									placeholder="Username"
									maxlength="128"
									autocomplete="off"
									spellcheck="false"
									:aria-invalid="showAuthenticationError"
									@input="onFieldInput('username')"
									@blur="saveImmediately"
								>
							</div>
							<div class="form-field">
								<label for="proxy-password">Password</label>
								<input
									id="proxy-password"
									type="password"
									v-model="form.password"
									placeholder="Password"
									maxlength="256"
									autocomplete="off"
									:aria-invalid="showAuthenticationError"
									@input="onFieldInput('password')"
									@blur="saveImmediately"
								>
							</div>
						</div>
						<span v-if="showAuthenticationError" class="field-error">
							Enter a login before the password.
						</span>
					</div>
				</transition>

				<div v-if="saveStatus === 'error'" class="save-error" role="alert">
					Could not save the proxy. Try again.
				</div>

				<button
					v-if="hasSelectedAssignment"
					class="button remove-proxy"
					type="button"
					:disabled="saveStatus === 'removing'"
					@click="removeProxy"
				>
					{{ saveStatus === 'removing' ? 'Removing…' : 'Remove proxy' }}
				</button>
			</div>
		</div>
	</div>
</template>

<script>
const STORAGE_KEY = 'proxifiedContainersKey'
const SAVE_DELAY_MS = 350

const PROTOCOLS = [
	{
		value: 'http',
		label: 'HTTP',
		portPlaceholder: '8080',
		supportsAuthentication: true,
	},
	{
		value: 'https',
		label: 'HTTPS',
		portPlaceholder: '443',
		supportsAuthentication: true,
	},
	{
		value: 'socks4',
		label: 'SOCKS4',
		portPlaceholder: '1080',
		supportsAuthentication: false,
	},
	{
		value: 'socks',
		label: 'SOCKS5',
		portPlaceholder: '1080',
		supportsAuthentication: true,
	},
]

function blankForm() {
	return {
		type: 'http',
		host: '',
		port: '',
		username: '',
		password: '',
	}
}

function normalizeAssignments(assignments) {
	if (!Array.isArray(assignments)) {
		return []
	}

	return assignments.reduce((normalized, assignment) => {
		if (
			!assignment ||
			typeof assignment.cookieStoreId !== 'string' ||
			!assignment.proxy
		) {
			return normalized
		}

		normalized.push({
			cookieStoreId: assignment.cookieStoreId,
			enabled: assignment.enabled !== false,
			proxy: assignment.proxy,
		})
		return normalized
	}, [])
}

function normalizeHost(host) {
	const trimmed = typeof host === 'string' ? host.trim() : ''
	if (trimmed.startsWith('[') && trimmed.endsWith(']')) {
		return trimmed.slice(1, -1)
	}
	return trimmed
}

function isValidIpv4(host) {
	const octets = host.split('.')
	return (
		octets.length === 4 &&
		octets.every((octet) => (
			/^\d{1,3}$/.test(octet) &&
			Number(octet) >= 0 &&
			Number(octet) <= 255
		))
	)
}

function isValidIpv6(host) {
	if (!host.includes(':')) {
		return false
	}

	try {
		const parsed = new URL(`http://[${host}]/`)
		return parsed.hostname.length > 2
	} catch (error) {
		return false
	}
}

function isValidHostname(host) {
	if (host.length > 253 || host.includes('..')) {
		return false
	}

	const labels = host.split('.')
	return labels.every((label) => (
		label.length >= 1 &&
		label.length <= 63 &&
		/^[a-z0-9](?:[a-z0-9-]*[a-z0-9])?$/i.test(label)
	))
}

function isValidProxyHost(host) {
	if (!host || /[\s/?#@]/.test(host)) {
		return false
	}

	if (host.includes(':')) {
		return isValidIpv6(host)
	}

	if (/^[\d.]+$/.test(host)) {
		return isValidIpv4(host)
	}

	return isValidHostname(host)
}

function replaceAssignment(assignments, cookieStoreId, proxy) {
	const normalized = normalizeAssignments(assignments)
	const existing = normalized.find((assignment) => (
		assignment.cookieStoreId === cookieStoreId
	))
	const remaining = normalized.filter((assignment) => (
		assignment.cookieStoreId !== cookieStoreId
	))

	remaining.push({
		cookieStoreId,
		enabled: existing ? existing.enabled : true,
		proxy,
	})
	return remaining
}

export default {
	data() {
		return {
			view: 'containers',
			protocols: PROTOCOLS,
			proxies: [],
			contextualIdentities: [],
			selectedCookieStoreId: '',
			form: blankForm(),
			touched: {
				host: false,
				port: false,
				username: false,
				password: false,
			},
			saveStatus: 'idle',
			saveTimer: null,
			saveRevision: 0,
			saveQueue: Promise.resolve(),
			lastSavedSignature: '',
			loadError: '',
			toggleError: '',
			togglePendingCookieStoreIds: [],
		}
	},
	computed: {
		containerRows() {
			return this.contextualIdentities.map((container) => {
				const assignment = this.proxies.find((item) => (
					item.cookieStoreId === container.cookieStoreId
				))
				return {
					...container,
					hasProxy: Boolean(assignment),
					proxyEnabled: Boolean(assignment && assignment.enabled),
				}
			})
		},
		selectedContainer() {
			return this.contextualIdentities.find((container) => (
				container.cookieStoreId === this.selectedCookieStoreId
			)) || null
		},
		selectedContainerName() {
			return this.selectedContainer ? this.selectedContainer.name : ''
		},
		selectedAssignment() {
			return this.proxies.find((assignment) => (
				assignment.cookieStoreId === this.selectedCookieStoreId
			)) || null
		},
		hasSelectedAssignment() {
			return Boolean(this.selectedAssignment)
		},
		selectedProtocol() {
			return this.protocols.find((protocol) => (
				protocol.value === this.form.type
			)) || this.protocols[0]
		},
		supportsAuthentication() {
			return this.selectedProtocol.supportsAuthentication
		},
		portPlaceholder() {
			return this.selectedProtocol.portPlaceholder
		},
		normalizedHost() {
			return normalizeHost(this.form.host)
		},
		hostIsValid() {
			return isValidProxyHost(this.normalizedHost)
		},
		portNumber() {
			return Number(this.form.port)
		},
		portIsValid() {
			return (
				/^\d+$/.test(this.form.port) &&
				Number.isInteger(this.portNumber) &&
				this.portNumber >= 1 &&
				this.portNumber <= 65535
			)
		},
		authenticationIsValid() {
			return !(
				this.supportsAuthentication &&
				this.form.password.length > 0 &&
				this.form.username.length === 0
			)
		},
		canPersistProxy() {
			return (
				this.selectedCookieStoreId.length > 0 &&
				this.hostIsValid &&
				this.portIsValid &&
				this.authenticationIsValid
			)
		},
		showHostError() {
			return this.touched.host && !this.hostIsValid
		},
		showPortError() {
			return this.touched.port && !this.portIsValid
		},
		showAuthenticationError() {
			return (
				(this.touched.username || this.touched.password) &&
				!this.authenticationIsValid
			)
		},
		normalizedProxy() {
			const proxy = {
				type: this.form.type,
				host: this.normalizedHost,
				port: this.portNumber,
			}

			if (this.supportsAuthentication && this.form.username.length > 0) {
				proxy.username = this.form.username
				proxy.password = this.form.password
			}

			return proxy
		},
		currentSignature() {
			return this.canPersistProxy
				? JSON.stringify(this.normalizedProxy)
				: ''
		},
		proxyPreview() {
			if (!this.canPersistProxy) {
				return ''
			}

			const displayHost = this.normalizedHost.includes(':')
				? `[${this.normalizedHost}]`
				: this.normalizedHost
			return `${this.selectedProtocol.label} · ${displayHost}:${this.portNumber}`
		},
	},
	async mounted() {
		try {
			const [stored, containers] = await Promise.all([
				browser.storage.local.get({ [STORAGE_KEY]: [] }),
				browser.contextualIdentities.query({}),
			])
			this.proxies = normalizeAssignments(stored[STORAGE_KEY])
			this.contextualIdentities = containers
		} catch (error) {
			console.debug('Failed to load proxy settings:', error)
			this.loadError = 'Could not load Firefox containers.'
		}

		browser.storage.onChanged.addListener(this.syncStorage)
		browser.contextualIdentities.onCreated.addListener(this.refreshContainers)
		browser.contextualIdentities.onUpdated.addListener(this.refreshContainers)
		browser.contextualIdentities.onRemoved.addListener(this.refreshContainers)
	},
	beforeUnmount() {
		if (this.saveTimer) {
			clearTimeout(this.saveTimer)
			this.saveTimer = null
			this.saveImmediately()
		}
	},
	unmounted() {
		browser.storage.onChanged.removeListener(this.syncStorage)
		browser.contextualIdentities.onCreated.removeListener(this.refreshContainers)
		browser.contextualIdentities.onUpdated.removeListener(this.refreshContainers)
		browser.contextualIdentities.onRemoved.removeListener(this.refreshContainers)
	},
	methods: {
		tooltipText(value, threshold) {
			if (
				typeof value !== 'string' ||
				typeof threshold !== 'number' ||
				value.length <= threshold
			) {
				return ''
			}
			return value
		},
		isProxyTogglePending(cookieStoreId) {
			return this.togglePendingCookieStoreIds.includes(cookieStoreId)
		},
		proxyToggleLabel(container) {
			return `Proxy for ${container.name}`
		},
		proxyToggleTitle(container) {
			return container.proxyEnabled
				? 'Temporarily disable proxy'
				: 'Enable proxy'
		},
		async toggleProxy(container) {
			if (
				!container.hasProxy ||
				this.isProxyTogglePending(container.cookieStoreId)
			) {
				return
			}

			const cookieStoreId = container.cookieStoreId
			const enabled = !container.proxyEnabled
			const previousAssignments = this.proxies
			this.toggleError = ''
			this.togglePendingCookieStoreIds.push(cookieStoreId)
			this.proxies = this.proxies.map((assignment) => (
				assignment.cookieStoreId === cookieStoreId
					? { ...assignment, enabled }
					: assignment
			))

			const operation = this.saveQueue
				.catch(() => {})
				.then(async () => {
					const stored = await browser.storage.local.get({ [STORAGE_KEY]: [] })
					const assignments = normalizeAssignments(stored[STORAGE_KEY])
					if (!assignments.some((assignment) => (
						assignment.cookieStoreId === cookieStoreId
					))) {
						return assignments
					}

					const updated = assignments.map((assignment) => (
						assignment.cookieStoreId === cookieStoreId
							? { ...assignment, enabled }
							: assignment
					))
					await browser.storage.local.set({
						[STORAGE_KEY]: updated,
					})
					return updated
				})

			this.saveQueue = operation

			try {
				this.proxies = await operation
			} catch (error) {
				console.debug('Failed to change the container proxy state:', error)
				this.proxies = previousAssignments
				this.toggleError = 'Could not change the proxy state. Try again.'
			} finally {
				this.togglePendingCookieStoreIds = this.togglePendingCookieStoreIds
					.filter((id) => id !== cookieStoreId)
			}
		},
		openContainer(cookieStoreId) {
			this.selectedCookieStoreId = cookieStoreId
			const assignment = this.proxies.find((item) => (
				item.cookieStoreId === cookieStoreId
			))
			const proxy = assignment && assignment.proxy

			this.form = proxy
				? {
					type: PROTOCOLS.some((protocol) => protocol.value === proxy.type)
						? proxy.type
						: 'http',
					host: typeof proxy.host === 'string' ? proxy.host : '',
					port: proxy.port ? String(proxy.port) : '',
					username: typeof proxy.username === 'string' ? proxy.username : '',
					password: typeof proxy.password === 'string' ? proxy.password : '',
				}
				: blankForm()

			this.touched = {
				host: false,
				port: false,
				username: false,
				password: false,
			}
			this.lastSavedSignature = proxy ? JSON.stringify(this.normalizedProxy) : ''
			this.saveStatus = proxy ? 'saved' : 'idle'
			this.view = 'proxy'
		},
		backToContainers() {
			this.saveImmediately()
			this.view = 'containers'
			this.selectedCookieStoreId = ''
			this.form = blankForm()
			this.lastSavedSignature = ''
			this.saveStatus = 'idle'
		},
		onProtocolChanged() {
			if (!this.supportsAuthentication) {
				this.form.username = ''
				this.form.password = ''
			}
			this.onFieldInput('type')
		},
		onFieldInput(field) {
			if (Object.prototype.hasOwnProperty.call(this.touched, field)) {
				this.touched[field] = true
			}

			if (this.saveTimer) {
				clearTimeout(this.saveTimer)
				this.saveTimer = null
			}

			if (!this.canPersistProxy) {
				this.saveStatus = 'incomplete'
				return
			}

			if (this.currentSignature === this.lastSavedSignature) {
				this.saveStatus = 'saved'
				return
			}

			this.saveStatus = 'pending'
			this.saveTimer = setTimeout(() => {
				this.saveTimer = null
				this.persistProxy()
			}, SAVE_DELAY_MS)
		},
		saveImmediately() {
			if (this.saveTimer) {
				clearTimeout(this.saveTimer)
				this.saveTimer = null
			}

			if (
				this.canPersistProxy &&
				this.currentSignature !== this.lastSavedSignature
			) {
				this.persistProxy()
			}
		},
		async persistProxy() {
			if (!this.canPersistProxy) {
				return
			}

			const cookieStoreId = this.selectedCookieStoreId
			const proxy = { ...this.normalizedProxy }
			const signature = JSON.stringify(proxy)
			const revision = ++this.saveRevision
			this.saveStatus = 'saving'

			const operation = this.saveQueue
				.catch(() => {})
				.then(async () => {
					const stored = await browser.storage.local.get({ [STORAGE_KEY]: [] })
					const updated = replaceAssignment(
						stored[STORAGE_KEY],
						cookieStoreId,
						proxy,
					)
					await browser.storage.local.set({
						[STORAGE_KEY]: updated,
					})
					return updated
				})

			this.saveQueue = operation

			try {
				const updated = await operation
				this.proxies = updated

				if (
					revision === this.saveRevision &&
					cookieStoreId === this.selectedCookieStoreId &&
					signature === this.currentSignature
				) {
					this.lastSavedSignature = signature
					this.saveStatus = 'saved'
				}
			} catch (error) {
				console.debug('Failed to save the container proxy:', error)
				if (revision === this.saveRevision) {
					this.saveStatus = 'error'
				}
			}
		},
		async removeProxy() {
			if (!this.selectedCookieStoreId) {
				return
			}

			if (this.saveTimer) {
				clearTimeout(this.saveTimer)
				this.saveTimer = null
			}

			const cookieStoreId = this.selectedCookieStoreId
			const revision = ++this.saveRevision
			this.saveStatus = 'removing'

			const operation = this.saveQueue
				.catch(() => {})
				.then(async () => {
					const stored = await browser.storage.local.get({ [STORAGE_KEY]: [] })
					const updated = normalizeAssignments(stored[STORAGE_KEY])
						.filter((assignment) => (
							assignment.cookieStoreId !== cookieStoreId
						))
					await browser.storage.local.set({
						[STORAGE_KEY]: updated,
					})
					return updated
				})

			this.saveQueue = operation

			try {
				const updated = await operation
				this.proxies = updated

				if (
					revision === this.saveRevision &&
					cookieStoreId === this.selectedCookieStoreId
				) {
					this.form = blankForm()
					this.touched = {
						host: false,
						port: false,
						username: false,
						password: false,
					}
					this.lastSavedSignature = ''
					this.saveStatus = 'removed'
				}
			} catch (error) {
				console.debug('Failed to remove the container proxy:', error)
				if (revision === this.saveRevision) {
					this.saveStatus = 'error'
				}
			}
		},
		syncStorage(changes, areaName) {
			if (areaName === 'local' && changes[STORAGE_KEY]) {
				this.proxies = normalizeAssignments(changes[STORAGE_KEY].newValue)
			}
		},
		async refreshContainers() {
			try {
				this.contextualIdentities = await browser.contextualIdentities.query({})
				if (
					this.selectedCookieStoreId &&
					!this.contextualIdentities.some((container) => (
						container.cookieStoreId === this.selectedCookieStoreId
					))
				) {
					this.view = 'containers'
					this.selectedCookieStoreId = ''
					this.form = blankForm()
				}
			} catch (error) {
				console.debug('Failed to refresh Firefox containers:', error)
			}
		},
	},
}
</script>

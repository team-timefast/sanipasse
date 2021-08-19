<script lang="ts">
	import type { CommonCertificateInfo } from '$lib/common_certificate_info';
	import { findCertificateError, parse_any } from '$lib/detect_certificate';
	import { assets } from '$app/paths';
	import type { ConfigProperties, HTTPRequest } from './_config';
	import QrCodeVideoReader from '../_QrCodeVideoReader.svelte';
	import { onDestroy } from 'svelte';

	export let config: ConfigProperties;
	const { decode_after_s, reset_after_s, prevent_revalidation_before_minutes } = config;

	let code: string = '';
	let codeFoundPromise: Promise<CommonCertificateInfo> | undefined = undefined;

	//let timeout: NodeJS.Timeout | undefined = undefined;
	let reset_timeout: NodeJS.Timeout | undefined = undefined;
	let scanning = false;

	interface PassHistory {
		first_name: string;
		last_name: string;
		date_of_birth: Date;
		validated: boolean;
		visible: boolean;
		date_of_validation: Date;
	}

	type PassHistoryEntry = PassHistory;

	let last_event: KeyboardEvent | null = null;
	let validated_passes: Map<string, number> = new Map();
	let passes_history: PassHistoryEntry[] = [];
	const pass_history_clean_interval = setInterval(() => {
		passes_history = passes_history.filter((passHistoryEntry) => Date.now() - passHistoryEntry.date_of_validation.getTime() < 30000);
	}, 5000);

	const prevent_revalidation_before_ms = (prevent_revalidation_before_minutes || 0) * 60 * 1000;

	function onKeyPress(event: KeyboardEvent) {
		last_event = event;
		if (event.key.length > 1) {
			launchParsing(code);
			//code = '';
			return;
		}
		if(scanning == false) {
			scanning = true;
			code = '';
			if(codeFoundPromise) {
				codeFoundPromise = undefined;
			}
		}
		code += event.key;
		//if (timeout !== undefined) clearTimeout(timeout);
		if (reset_timeout !== undefined) {
			clearTimeout(reset_timeout);
			passes_history = passes_history.map(ph => Object.assign(ph, {visible: true}));
		}
		//timeout = setTimeout(launchParsing, decode_after_s * 1000, code);
		event.preventDefault();
	}

	function onPaste({ clipboardData }: ClipboardEvent) {
		if (!clipboardData) return;
		launchParsing(clipboardData.getData('text'));
	}

	async function validateCertificateCode(code: string): Promise<CommonCertificateInfo> {
		const cert = await parse_any(code);
		const error = findCertificateError(cert);

		if (error) {
                        return {cert, error: new Error(error)};
		}
		const last_validated = validated_passes.get(code);

		if (last_validated && last_validated > Date.now() - prevent_revalidation_before_ms) {
			const duration_minutes = ((Date.now() - last_validated) / 60 / 1000) | 0;
                        return {cert, error: new Error(
                                `Passe déjà scanné par quelqu'un d'autre il y a ` +
                                        (duration_minutes ? duration_minutes + ' minutes.' : "moins d'une minute.")
                        )};
		}
		validated_passes.set(code, Date.now());
		return {cert};
	}

	async function makeRequest(r: HTTPRequest) {
		return fetch(r.url, { method: r.method, body: r.body || undefined });
	}

	async function onValid() {
		if (config.external_requests && config.external_requests.accepted.url)
			return makeRequest(config.external_requests.accepted);
	}

	async function onInvalid() {
		if (config.external_requests && config.external_requests.refused.url)
			return makeRequest(config.external_requests.refused);
	}

	function launchParsing(code_input: string) {
		if (codeFoundPromise) return;
		console.log('Detected code before reset: ', code_input);
		codeFoundPromise = validateCertificateCode(code_input);
		codeFoundPromise.then(({error, cert}) => {
			let data: PassHistoryEntry = {
				first_name: cert.first_name,
				last_name: cert.last_name,
				date_of_birth: cert.date_of_birth,
				validated: false,
				visible: false,
				date_of_validation: new Date(Date.now())
			};
			if(error == undefined) {
				data = Object.assign(data, {validated: true});
				onValid();
			} else {
				data = Object.assign(data, {validated: false});
				onInvalid();
			}
			passes_history = [data, ...passes_history].slice(0, 4);
			reset_timeout = setTimeout(() => {
				codeFoundPromise = undefined;
				passes_history[0] = Object.assign(passes_history[0], {visible: true});
			}, reset_after_s * 1000);
		});
		//timeout = undefined;
		code = '';
		scanning = false;
	}

	function showName({ first_name, last_name, date_of_birth }: CommonCertificateInfo): string {
		return (
			(first_name[0] || '').toUpperCase() +
			first_name.slice(1).toLowerCase() +
			' ' +
			last_name.toUpperCase() +
			' né(e) le ' +
			date_of_birth.toLocaleDateString("fr")
		);
	}

	onDestroy(() => clearInterval(pass_history_clean_interval));
</script>

<svelte:window on:keypress={onKeyPress} on:paste={onPaste} />

<div
	class="main container"
	style="font-family: {config.font || 'inherit'}; font-size: {config.font_size || 16}px"
>

	{#if scanning !== false}
		Scan du QR code en cours...
	{:else if codeFoundPromise != undefined}
		{#await codeFoundPromise}
			Décodage du code...
		{:then passOrErr}
                    {#if passOrErr.error == undefined}
			<!-- svelte-ignore a11y-media-has-caption -->
			<audio autoplay src="{assets}/valid.mp3" />
			<div class="alert alert-success" role="alert">
				<div class="row">
					<div class="col-md-2"><div class="sign shallpass" /></div>
					<div class="col-md-10">
						<h3>
							Bienvenue, {#if !config.anonymize}{showName(passOrErr.cert)}{/if}
						</h3>
						<p>Votre passe est validé.</p>
						<div class="progress">
							<div
								class="progress-bar bg-success animate"
								role="progressbar"
								style="animation-duration: {reset_after_s}s"
							/>
						</div>
					</div>
				</div>
			</div>
		    {:else}
			<!-- svelte-ignore a11y-media-has-caption -->
			<audio autoplay src="{assets}/invalid.mp3" />
			<div class="alert alert-danger" role="alert">
				<div class="row">
					<div class="col-md-2"><div class="sign shallnotpass" /></div>
					<div class="col-md-10">
						<h3>Passe sanitaire invalide</h3>
						<p class="font-monospace">{passOrErr.error.message}</p>
						<div class="progress">
							<div
								class="progress-bar bg-danger animate"
								role="progressbar"
								style="animation-duration: {reset_after_s}s"
							/>
						</div>
					</div>
				</div>
			</div>
                    {/if}
		{/await}
	{:else}
		<div class="row justify-content-center w-100">
			{#each config.logo_urls as url}
				<img alt="logo" src={url} class="col" style="object-fit: contain; max-height: 10em;" />
			{/each}
		</div>

		<h1>{config.title}</h1>
		<p>{config.description}</p>
	{/if}

	{#each passes_history as passHistoryEntry}
		{#if passHistoryEntry.visible}
			<div class="alert {passHistoryEntry.validated ? 'alert-success' : 'alert-danger'}" role="alert">
					<div class="row">
							<div class="col-md-2"><div class="sign {passHistoryEntry.validated ? 'shallpass' : 'shallnotpass'}" /></div>
							<div class="col-md-10">
									<h3>{showName(passHistoryEntry)}</h3>
									<p class="font-monospace">Date du scan : {passHistoryEntry.date_of_validation.toLocaleString("fr")}</p>
							</div>
					</div>
			</div>
		{/if}
	{/each}

	{#if config.video_scan}
		<div class="videoinput w-100" style="display: {codeFoundPromise ? 'none' : 'flex'}">
			<QrCodeVideoReader
				on:qrcode={({ detail }) => launchParsing(detail)}
				facingMode={config.video_facing_mode}
			/>
		</div>
	{/if}

	{#if config.debug}
		<div>
			Code:
			<p class="text-break font-monospace">{code}</p>
			<p>Code length: {code.length}</p>
			<p>Last key pressed: {JSON.stringify(last_event?.key, null, ' ')}</p>
			<p>Scanning: {scanning ? 'yes' : 'no'}</p>
			<p>Historique : {JSON.stringify(passes_history)}</p>
		</div>
	{/if}

	<p class="fixed-bottom text-muted fw-lighter fst-italic" style="font-size: .8em">
		{config.bottom_infos}
	</p>
</div>

<style>
	.progress-bar {
		width: 100%;
		transition: 100ms;
	}

	.progress-bar.animate {
		animation: reduce_width;
	}

	@keyframes reduce_width {
		to {
			width: 0%;
		}
	}

	.sign {
		border: 0.1em solid white;
		width: 5em;
		height: 5em;
		border-radius: 50%;
		display: flex;
		align-items: center;
		justify-content: center;
	}

	.shallnotpass {
		background-color: var(--bs-danger);
	}
	.shallnotpass::after {
		content: ' ';
		display: block;
		background-color: white;
		animation: shallnotpass 0.5s;
		height: 0.8em;
		width: 65%;
	}
	@keyframes shallnotpass {
		from {
			height: 0;
			width: 0%;
		}
		to {
			height: 0.8em;
			width: 65%;
		}
	}

	.shallpass {
		background-color: var(--bs-success);
		animation: shallpass 0.8s;
	}
	.shallpass::before {
		content: ' ';
		display: block;
		background-color: white;
		height: 1em;
		width: 0.8em;
		transform: rotate(-45deg) translate(0.37em, 0.4em);
	}
	.shallpass::after {
		content: ' ';
		display: block;
		background-color: white;
		height: 0.8em;
		width: 50%;
		transform: rotate(-45deg);
	}
	@keyframes shallpass {
		from {
			transform: rotate(360deg);
		}
		to {
			transform: rotate(0);
		}
	}
	.videoinput {
		max-height: 45vh;
		display: flex;
		justify-content: center;
	}
	h1 {
		font-size: 2em;
	}
</style>

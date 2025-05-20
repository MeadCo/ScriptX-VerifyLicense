<script>
	import { onMount } from 'svelte';

	let url = '';
	let loading = false;
	let valid = false;
	let connected = false;
	let secureWareHouseSupported = true;
	let loadedLicense = {
	revision: 0,
	from: "",
	to: "",
	company: "",
	guid: ""
	};
	let serviceVersion = " [.connecting..]";

	onMount(async () => {
	console.log("onMount connect lite....");
	try {
	MeadCo.ScriptX.Print.connectLite("http://127.0.0.1:41191","");
	// this only works anonymously for v2.21.1.16 and later (22.05.2024)
	serviceVersion = await new Promise((resolve,reject) => {
	MeadCo.ScriptX.Print.serviceVersionAsync(resolve,reject);
	});
	serviceVersion = serviceVersion.major + "." + serviceVersion.minor + "." + serviceVersion.revision;
	console.log("serviceVersion", serviceVersion);
	connected = true;
	} catch (error) {
	// if the error is unauthorised then service is running but is prior to v2.21.1.16
	console.log("error on connect: ", error);
	if (typeof error === "string" && error.toLowerCase().includes("unauthorized")) {
	serviceVersion = " [.available.]";
	connected = true;
	secureWareHouseSupported = false;
	} else {
	alert(error);
	serviceVersion = " [.failed..]";
	}
	}
	});

	function isValidGuid(guid) {
	const guidRegex = /^[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}$/;
	return guidRegex.test(guid);
	}

	function IsHttps() {
	if (typeof window !== 'undefined') {
	return window.location.protocol === 'https:';
	}
	return false;
	}

	function isValidUrl(value) {
	try {
	if (value.toLowerCase() === 'warehouse' || value.toLowerCase() === 'securewarehouse') {
	if ( IsHttps() && secureWareHouseSupported) {
return "securewarehouse";
}
return value;
}
const url = new URL(value);
return url;
} catch (_) {
return "";
}
}

async function displayLicense(licenseGuid, path) {
console.log("start verify");
loading = true;
valid = false;
try {
let license = await new Promise((resolve,reject) => {
MeadCo.ScriptX.Print.Licensing.applyAsync(licenseGuid, 0, path, resolve, reject)
});

console.log('License details:', license);

// Store the full license object
loadedLicense = license;
loadedLicense.from = loadedLicense.from.split('T')[0];
loadedLicense.to = loadedLicense.to.split('T')[0];
loadedLicense.url = path;

valid = true;
} catch (error) {
valid = false;
alert('An error occurred while verifying the license: ' + error);
console.error(error);
}
finally {
console.log("end verify");
loading = false;
}
}

function handleVerify() {
if (!isValidGuid(loadedLicense.guid)) {
alert('Please enter a valid GUID for the license.');
return;
}
url = isValidUrl(url);
if (url === "") {
alert('Please enter "warehouse" or a valid URL.');
return;
}
displayLicense(loadedLicense.guid, url);
}

function handleDetails() {
if (typeof window !== 'undefined') {
sessionStorage.setItem('licenseData', JSON.stringify(loadedLicense));
window.location.href = 'print.html';
}
}
</script>

<div class="card">
	<div class="header">
		<h2>MeadCo ScriptX License Verification
			<a href="https://github.com/MeadCo/ScriptX-VerifyLicense" target="_blank" rel="noopener noreferrer" style="margin-left: 0.5em;">
				<img src="https://img.shields.io/badge/GitHub-Repo-blue?logo=github" alt="GitHub Repo" style="vertical-align: middle;"/>
			</a>
		</h2>
		<h3>v1.0.9 using ScriptX Services for Windows PC v{serviceVersion}</h3>
	</div>

	<div class="form-group with-label">
		<label class="group-label" for="license-input">GUID *</label>
		<input id="license-input" type="text" bind:value={loadedLicense.guid} placeholder="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" />
	</div>

	<div class="form-group with-label">
		<label class="group-label" for="url-input">Url *</label>
		<input id="url-input" type="text" bind:value={url} placeholder="Warehouse or URI" />
	</div>

	<div class="header">
		<h2>License Summary</h2>
	</div>

	<div class="issued-to" class:loading>
		Issued to: <strong>{loadedLicense.company || "Not verified"}</strong>
	</div>
	
	<div class="form-row" class:loading>
		<div class="form-group with-label">
			<label class="group-label" for="revision-input">Revision</label>
			<input readonly id="revision-input" type="number" bind:value={loadedLicense.revision} />
		</div>
		
		<div class="form-group with-label">
			<label class="group-label" for="valid-from-input">Valid From</label>
			<input readonly id="valid-from-input" type="date" bind:value={loadedLicense.from} />
		</div>

		<div class="form-group with-label">
			<label class="group-label" for="valid-until-input">Valid Until</label>
			<input readonly id="valid-until-input" type="date" bind:value={loadedLicense.to} />
		</div>
	</div>

	<!--<div class="button-row">
		<button class="green" on:click={() => handleClick('general')} disabled={loading}>{loading ? 'Verifying...' : 'Verify license'}</button>
	</div>-->

	<div class="button-row">
		<button class="green verify-button" on:click={handleVerify} disabled={loading || !connected}>
			{loading ? 'Verifying...' : 'Verify license'}
		</button>
		<button class="blue print-button" on:click={handleDetails} disabled={!valid || !connected}>
			Details ...
		</button>
	</div>

</div>

<!-- Required to make the route discoverable to prerendering -->
<a href="print" style="display:none">Print</a>


<!-- Add this after the card div closing tag -->
{#if loading}
<div class="overlay">
	<div class="spinner"></div>
	<div class="loading-text">Verifying license...</div>
</div>
{/if}

<style>
	.card {
	max-width: 540px;
	margin: 2rem auto;
	padding: 1.5rem;
	background: white;
	border-radius: 8px;
	box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
	font-family: sans-serif;
	}

	.header {
	margin-bottom: 1rem;
	}

	.header h2 {
	margin-bottom: 0.0rem;
	}

	.header h3 {
	margin-top: 0.25rem;
	font-size: 0.8rem;
	margin-bottom: 1.5rem;
	}

	.form-group {
	margin-bottom: 1rem;
	display: flex;
	flex-direction: column;
	}

	.form-group label {
	margin-bottom: 0.25rem;
	font-weight: bold;
	font-size: 0.85rem; /* Slightly reduce font size */
	}

	.form-group input[type="number"],
	.form-group input[type="text"],
	.form-group input[type="date"] {
	padding: 0.25rem;
	font-size: 0.9rem; /* Reduce font size */
	width: 100%; /* Ensure inputs use full width */
	box-sizing: border-box; /* Include padding in width calculation */
	}

	/* Right align number input fields */
	.form-group input[type="number"] {
	text-align: right;
	padding-right: 0.5rem;
	}

	.form-group.with-label {
	border: 1px solid #ccc;
	border-radius: 6px;
	padding: 1rem 0.5rem 0.5rem 0.5rem;
	position: relative;
	}

	.form-group.with-label input {
	border: none;
	outline: none;
	}

	.form-group.with-label .group-label {
	position: absolute;
	top: -0.5rem;
	left: 0.75rem;
	background: white;
	padding: 0 0.25rem;
	font-size: 0.85rem;
	color: #333;
	font-weight: bold;
	}

	.form-row {
	display: flex;
	gap: 0.5rem;
	flex-wrap: wrap; /* Allow wrapping on small screens */
	width: 100%;
	}

	.form-row .form-group {
	flex: 1;
	min-width: 25%; /* Set a minimum width */
	}

	.button-row {
	display: flex;
	justify-content: space-between;
	margin-top: 1.5rem;
	width: 100%;
	gap: 0.75rem;
	}

	.button-row button {
	padding: 0.75rem;
	border: none;
	border-radius: 4px;
	font-size: 1rem;
	font-weight: bold;
	cursor: pointer;
	}

	.verify-button {
	flex: 3;
	}

	.print-button {
	flex: 1;
	}

	.green {
	background-color: #43a047;
	color: white;
	}

	.green:hover:not(:disabled) {
	background-color: #388e3c;
	}

	.blue {
	background-color: #1976d2;
	color: white;
	}

	.blue:hover {
	background-color: #1565c0;
	}

	/* Add responsive adjustments for small screens */
	@media (max-width: 480px) {
	.form-row {
	flex-direction: column;
	}

	.form-row .form-group {
	width: 100%;
	}
	}

	.issued-to {
	padding: 0.75rem 1rem;
	margin-bottom: 1rem;
	border: 1px solid #e0e0e0;
	border-radius: 6px;
	background-color: #f5f5f5;
	font-size: 0.9rem;
	color: #333;
	}

	.issued-to strong {
	color: #1a1a1a;
	margin-left: 0.25rem;
	}

	.button-row button:disabled {
	background-color: #9e9e9e;  /* Grey color for disabled buttons */
	color: #e0e0e0;  /* Light grey text for disabled buttons */
	cursor: not-allowed;
	opacity: 0.7;
	}

	.print-button:disabled {
	background-color: #9e9e9e !important;  /* Grey color that overrides the blue */
	}

	.overlay {
	position: fixed;
	top: 0;
	left: 0;
	right: 0;
	bottom: 0;
	background-color: rgba(0, 0, 0, 0.5);
	display: flex;
	flex-direction: column;
	justify-content: center;
	align-items: center;
	z-index: 1000;
	}

	.spinner {
	width: 40px;
	height: 40px;
	border: 4px solid #f3f3f3;
	border-top: 4px solid #43a047;
	border-radius: 50%;
	animation: spin 1s linear infinite;
	}

	.loading-text {
	margin-top: 1rem;
	color: white;
	font-name: Arial;
	font-size: 24px;
	font-weight: bold;
	}

	@keyframes spin {
	0% { transform: rotate(0deg); }
	100% { transform: rotate(360deg); }
	}

	/* Add visual effects to the license details section when loading */
	.form-row.loading {
	opacity: 0.5;
	pointer-events: none;
	transition: opacity 0.3s ease;
	}

	.issued-to.loading {
	opacity: 0.5;
	transition: opacity 0.3s ease;
	}

</style>

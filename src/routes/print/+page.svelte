<script>
    import { goto } from '$app/navigation';
    import { onMount } from 'svelte';

    let licenseData = null;
    let loading = true;
    let showDialog = false;
    let printers = [];
    let selectedPrinter = "";
    let defaultPrinter = "";
    const server = "http://127.0.0.1:41191";

    let printing = false;

    const optionLabels = {
        basicHtmlPrinting: "Basic HTML Printing",
        advancedPrinting: "Advanced Printing",
        enhancedFormatting: "Enhanced Formatting",
        printPdf: "PDF Printing",
        printRaw: "Direct Printing"
    };

    onMount(async () => {
        const printApi = MeadCo.ScriptX.Print;
        const licenseApi = MeadCo.ScriptX.Print.Licensing;

        // Retrieve the license data from sessionStorage
        const storedData = sessionStorage.getItem('licenseData');
        if (storedData) {
            licenseData = JSON.parse(storedData);

            licenseData.from = licenseData.from.split('T')[0];
            licenseData.to = licenseData.to.split('T')[0];

            // Clear the stored data to prevent access after refresh
            sessionStorage.removeItem('licenseData');
        }

        try {
            console.log('Initializing MeadCo ScriptX...', licenseData);
            licenseApi.connectLite(server, licenseData.guid, licenseData.revision, licenseData.url);
            printApi.connectLite(server, licenseData.guid);

            await MeadCo.ScriptX.InitAsync();

            printers = MeadCo.ScriptX.GetAvailablePrinters();
            defaultPrinter = MeadCo.ScriptX.Print.printerName;
            console.log('Available printers:', printers);
            console.log('Default printer:', defaultPrinter);

            // Set selectedPrinter to defaultPrinter if it exists in the list, otherwise use the first printer
            if (printers && printers.length > 0) {
                selectedPrinter = printers.includes(defaultPrinter) ? defaultPrinter : printers[0];
            }

            MeadCo.ScriptX.Printing.header = "";
            MeadCo.ScriptX.Printing.footer = "&b&p/&P&b&d";
            MeadCo.ScriptX.Printing.pageSize = "A4";
            MeadCo.ScriptX.Printing.orientation = "portrait";
            MeadCo.ScriptX.Printing.printBackground = false;

            MeadCo.ScriptX.Printing.SetMarginMeasure(1); // 1 = mm, 2 = inch
            MeadCo.ScriptX.Printing.topMargin = 10;
            MeadCo.ScriptX.Printing.bottomMargin = 10;
            MeadCo.ScriptX.Printing.leftMargin = 10;
            MeadCo.ScriptX.Printing.rightMargin = 10;

            loading = false;
        } catch (error) {
            console.error('Error initializing MeadCo ScriptX:', error);
        }
    });

    // Function to go back to the main page
    function goBack() {
        goto('/verify');
    }

    function openPrintDialog() {
        console.log('Opening print dialog');
        showDialog = true;
        document.getElementById('printDialog').showModal();
    }

    function closeDialog() {
        console.log('Closing dialog');
        showDialog = false;
        document.getElementById('printDialog').close();
    }

    // called by the print button on the dialog
    async function doPrint() {
        console.log('Printing to:', selectedPrinter);
        closeDialog();
        try {
            printing = true;

            // ensure we have the required information for the selected printer
            await new Promise((resolve, reject) => {
                MeadCo.ScriptX.Print.deviceSettingsForAsync(selectedPrinter, resolve, reject);
            });

            // can now select it
            MeadCo.ScriptX.Print.printerName = selectedPrinter;
            MeadCo.ScriptX.PrintPage(false);
            await MeadCo.ScriptX.WaitForSpoolingComplete();
        }
        catch (error) {
            console.error('Error during print:', error);
            alert('An error occurred while printing. Please check your printer settings.');
        }
        finally {
            printing = false;
        }
    }
</script>

<div class="print-container" role="main" aria-labelledby="license-title">
    <div class="print-header">
        <h1 id="license-title">MeadCo ScriptX License Information</h1>
        <div class="non-printable">
            <button class="back-button" on:click={goBack}>Back</button>
            <button class="print-button" on:click={openPrintDialog} disabled={loading}>{loading ? 'Please wait' : 'Print'}</button>
        </div>
    </div>

    {#if licenseData}
        <div class="license-details">
            <div class="detail-section company-section">
                <h2 class="license-h">License Holder</h2>
                <p class="company-name">{licenseData.company || 'Not specified'}</p>
            </div>

            <div class="detail-section">
                <h2 class="license-details-heading">License Details</h2>
                <div class="detail-grid" style="margin-bottom: 1rem">
                    <div class="detail-item row-item">
                        <span class="label">Downloaded from:</span>
                        <span class="value">{licenseData.url}</span>
                    </div>
                </div>
                <div class="detail-grid">
                    <div class="detail-item">
                        <span class="label">License ID:</span>
                        <span class="value">{licenseData.guid}</span>
                    </div>
                    <div class="detail-item">
                        <span class="label">Revision:</span>
                        <span class="value">{licenseData.revision}</span>
                    </div>
                    <div class="detail-item">
                        <span class="label">Valid From:</span>
                        <span class="value">{new Date(licenseData.from).toLocaleDateString()}</span>
                    </div>
                    <div class="detail-item">
                        <span class="label">Valid Until:</span>
                        <span class="value">{new Date(licenseData.to).toLocaleDateString()}</span>
                    </div>
                </div>

                <span class="label" style="margin-top:0.5rem;display:block;">License Options:</span>
                <div class="options-row">
                    {#each Object.entries(licenseData.options) as [key, value]}
                        <label class="option-checkbox">
                            <input type="checkbox" checked={value} disabled />
                            {optionLabels[key] || key}
                        </label>
                    {/each}
                </div>

                <span class="label" style="margin-bottom:0.5rem;display:block;">Valid Domains:</span>
                <ul class="domains-list">
                    {#each licenseData.domains as domain}
                        <li>{domain}</li>
                    {/each}
                </ul>
            </div>

            <div class="detail-section footer-section">
                <p>This license verification was performed on {new Date().toLocaleDateString()} at {new Date().toLocaleTimeString()}</p>
            </div>
        </div>
    {:else}
        <div class="no-data">
            <p>No license data available. Please verify a license first.</p>
            <button class="back-button" on:click={goBack}>Return to License Verification</button>
        </div>
    {/if}
</div>

<dialog id="printDialog" aria-labelledby="dialog-title" aria-describedby="dialog-desc">
    <form method="dialog" class="dialog-content">
        <h2 id="dialog-title">Print options</h2>
        <p id="dialog-desc" class="sr-only">Select a printer to use</p>        
        <div class="printer-select-container">
            <label for="printer-select" style="font-weight: bold;">Printer : </label>
            <select id="printer-select" bind:value={selectedPrinter} style="margin-left: 0.5rem;">
                {#each printers as printer}
                    <option value={printer}>{printer}</option>
                {/each}
            </select>
        </div>
        <div style="display: flex; justify-content: flex-end; gap: 0.5rem;">
            <button type="button" on:click={doPrint} class="print-button">Print</button>
            <button type="button" on:click={closeDialog} class="back-button">Cancel</button>
        </div>
    </form>
</dialog>

{#if printing}
  <div class="overlay" role="status" aria-live="polite">
    <div class="spinner" aria-hidden="true"></div>
    <div class="printing-text">Printing ...</div>
  </div>
{/if}

<style>
    .print-container {
        max-width: 800px;
        margin: 2rem auto;
        padding: 2rem;
        background: white;
        border-radius: 8px;
        box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
        font-family: "Segoe UI", Arial, sans-serif;
    }

    .print-header {
        display: flex;
        justify-content: space-between;
        align-items: center;
        margin-bottom: 2rem;
        border-bottom: 2px solid #43a047;
        padding-bottom: 1rem;
    }

    h1 {
        font-size: 1.8rem;
        color: #333;
        margin: 0;
    }

    h2 {
        font-size: 1.4rem;
        color: #43a047;
        margin-top: 0;
        margin-bottom: 1rem;
    }

    .dialog-content {
    min-width: 320px;
    }

    .printer-select-container {
    margin-bottom: 1rem;
    }

    .license-details-heading {
    margin-bottom: 0.1rem;
    }

    .detail-section {
        margin-bottom: 2rem;
        padding: 1.5rem;
        border: 1px solid #e0e0e0;
        border-radius: 6px;
        background-color: #f9f9f9;
    }

    .company-section {
        background-color: #f0f7f0;
        border-color: #c8e6c9;
    }

    .license-h {
        margin-bottom: 0rem;
    }

    .company-name {
        font-size: 1.6rem;
        font-weight: bold;
        color: #2e7d32;
        text-align: center;
        padding: 0;
        margin: 0;
    }

    .detail-grid {
        display: grid;
        grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
        gap: 1rem;
    }

    .detail-item {
        display: flex;
        flex-direction: column;
        margin-bottom: 0.5rem;
    }

    .detail-item.row-item {
        flex-direction: row;
        align-items: center;
        gap: 0.5rem;
    }

    .detail-item.row-item .label {
        font-weight: bold;
        margin-bottom: 0;
    }

    .detail-item.row-item .value {
        font-weight: normal;
        font-size: 0.9rem;
    }

    .label {
        font-weight: bold;
        font-size: 0.9rem;
        color: #555;
        margin-bottom: 0.25rem;
    }

    .value {
        font-size: 1.1rem;
        color: #333;
    }

    .options-row {
        display: flex;
        flex-wrap: wrap;
        gap: 0.5rem;
        align-items: center;
        margin-bottom: 1rem;
        margin-top: 1rem;
    }
    .option-checkbox {
        display: flex;
        align-items: center;
        font-family: inherit;
        font-size: 1rem;
        float: left;
        width: 33.33%;
        box-sizing: border-box;
        margin-bottom: 0.5em;
        /* For vertical alignment and spacing */
        padding-right: 1em;        
        gap: 0.4em;
        color: #222;
        user-select: none;
    }
    .option-checkbox input[type="checkbox"] {
        accent-color: #43a047;
        width: 1.1em;
        height: 1.1em;
        cursor: not-allowed;
        margin-right: 0.4em;        
    }

    .domains-list {
        list-style: none;
        margin: 0;
        padding-left: 1.2em;
        font-size: 1rem;
        color: #222;
    }
    .domains-list li {
        margin-bottom: 0.2em;
        font-family: inherit;
    }

    .footer-section {
        text-align: center;
        font-style: italic;
        color: #666;
        background-color: #f5f5f5;
    }

    .no-data {
        text-align: center;
        padding: 3rem;
        color: #666;
    }

    .back-button, .print-button {
        padding: 0.75rem 1.5rem;
        border: none;
        border-radius: 4px;
        font-size: 1rem;
        font-weight: bold;
        cursor: pointer;
        margin-left: 0.5rem;
    }

    .back-button {
        background-color: #757575;
        color: white;
    }

    .print-button {
        background-color: #1976d2;
        color: white;
    }

    .back-button:hover {
        background-color: #616161;
    }

    .print-button:hover {
        background-color: #1565c0;
    }

    .non-printable {
        display: flex;
    }

    dialog::backdrop {
        background: rgba(0,0,0,0.2);
    }

    dialog,
    dialog * {
        font-family: "Segoe UI Variable", "Segoe UI", system-ui, sans-serif;
        font-size: 1rem;
        color: #222;
    }

    dialog {
        border-radius: 8px;
        border: 1px solid #43a047;
        padding: 1.5rem 2rem;
        box-shadow: 0 2px 16px rgba(0,0,0,0.2);
    }
    dialog h2 {
        margin-top: 0;
        margin-bottom: 1rem;
        font-size: 1.2rem;
        color: #43a047;
    }

    #printer-select {
        font-family: "Segoe UI Variable", "Segoe UI", system-ui, sans-serif;
        font-size: 1rem;
        color: #1a1a1a;
        background: #f3f3f3;
        border: 1px solid #c9c9c9;
        border-radius: 6px;
        padding: 0.5rem 1.5rem 0.5rem 0.75rem;
        margin-left: 0.5rem;
        box-shadow: none;
        transition: border-color 0.2s, box-shadow 0.2s;
        appearance: none;
        -webkit-appearance: none;
        -moz-appearance: none;
        background-image: url("data:image/svg+xml;utf8,<svg fill='gray' height='16' viewBox='0 0 20 20' width='16' xmlns='http://www.w3.org/2000/svg'><path d='M5.516 7.548a1 1 0 0 1 1.415-.032L10 10.293l3.07-2.777a1 1 0 1 1 1.383 1.447l-3.777 3.414a1 1 0 0 1-1.383 0L5.548 8.963a1 1 0 0 1-.032-1.415z'/></svg>");
        background-repeat: no-repeat;
        background-position: right 0.75rem center;
        background-size: 1rem;
    }

    #printer-select:focus {
        outline: none;
        border-color: #c9c9c9;
        background: #fff;
        box-shadow: none;
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

    .printing-text {
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

    @media (max-width: 600px) {
        .print-container {
            margin: 1rem;
            padding: 1rem;
        }
        
        .detail-grid {
            display: block;
        }
    }    

    /* Print-specific styles */
    @media print {
        .print-container {
            margin: 0;
            padding: 0;
            box-shadow: none;
            max-width: 100%;
        }

        .print-header {
            display: -ms-flexbox;
            display: flex;
            -ms-flex-pack: justify;
            justify-content: space-between;
            -ms-flex-align: center;
            align-items: center;
            margin-bottom: 2rem;
            border-bottom: 2px solid #43a047;
            padding-bottom: 1rem;
        }

        .detail-grid {
            display: -ms-flexbox;
            display: flex;
            -ms-flex-wrap: wrap;
            flex-wrap: wrap;
            margin-bottom: 1rem;
        }
        .detail-item {
            display: -ms-flexbox;
            display: flex;
            -ms-flex-direction: column;
            flex-direction: column;
            margin-bottom: 0.5rem;
            min-width: 220px;
            margin-right: 1rem;
        }
        .detail-item.row-item {
            -ms-flex-align: baseline;
            align-items: baseline !important;
            gap: 0.5rem !important;
        }
        .detail-item.row-item .label {
            margin-right: 2.5em !important;
        }
        .detail-item.row-item .label,
        .detail-item.row-item .value {
            margin-bottom: 0;
            vertical-align: baseline;
            line-height: 1.2;
        }

        .options-row {
            display: -ms-flexbox;
            display: flex;
            -ms-flex-wrap: wrap;
            flex-wrap: wrap;
            gap: 2.5rem !important;
            align-items: center;
            margin-bottom: 1rem;
            margin-top: 1rem;
        }

        .option-checkbox {
            display: -ms-flexbox;
            display: flex;
            -ms-flex-align: center;
            align-items: center;
            font-family: inherit;
            font-size: 1rem;
            gap: 0.4em;
            color: #222;
            user-select: none;
            margin-right: 0 !important;
            float: left;
            width: 33.33%;            
            box-sizing: border-box;
            margin-bottom: 0.5em;
            padding-right: 1em;            
        }

        .option-checkbox input[type="checkbox"] {
            width: 1.1em;
            height: 1.1em;
            cursor: not-allowed;
            margin-right: 0.3em;
        }

        .non-printable {
            display: -ms-flexbox;
            display: none !important;
        }
        .detail-section {
            break-inside: avoid;
            page-break-inside: avoid;
        }
        .overlay, .spinner, dialog {
            display: none !important;
        }
    }
</style>
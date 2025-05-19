# ScriptX License Verification

ScriptX license Verification is a static HTML application developed with Svelte that checks a license is usable on a domain. It verifies if the license is valid, expired, or not found, and provides appropriate messages based on the verification results.

## Use

* Download the file 'app-vX.X.X.zip' from this repo and extract it to a folder named 'verify' at the root (/verify) on your server. 
* On a PC on which ScriptX.Services for Windows PC is installed, open the '/verify/index.html' file in a web browser to run the application.
* Enter the license guid and the url to the license file (typically lic.mlf) (or "warehouse" can be used) in the input fields and click the "Verify" button to check the license status.
* The application will display the verification result, including the license status (valid, expired, or not found) and the enabled options and domains associated with the license.
* Click the "print" button to print the verification result with the installed version of ScriptX.Services for Windows PC.

## Example

The evaluation license can be verified here:

https://support.meadroid.com/verify/

Use the evaluation license GUID. This can be found here:

https://support.meadroid.com/deploy/services/

## Customisation

To install the application in an alternative folder edit the svelte.config.js file and 
change the base path to the new folder. Also edit the function goBack() in the 
src/routes/print/+page.svelte file to point to the new folder. The default is /verify.



# ClearKey Content Protection (CCP)
Various solutions exist today to protect the distribution of DASH content over the internet
1.  TLS delivery: protects the unencrypted assets from interference and interception during transit using HTTPS. Does not implement any access control and media segments are unencrypted. 
2.  Token auth: requires the client to posses a time-delimited token to request the manifest and segments. The media content itself is unencrypted. 
3.  DRM: the segments are encrypted and the client must authenticate itself with a license server in order to obtain a decryption key. The decryption key is itself encrypted when it is transferred between client and license server. 

The cost of these solutions increases in the order described, with DRM providing the greatest level of protection but generally requiring the highest preparation, and in many cases licensing, cost.

Within the HLS content delivery ecosystem, there exists a protection mode known as "AES-128 Encryption" or "Envelope Encryption". With this mode, the entire media container is encrypted using AES-128 and a URL (often token protected and always over HTTPS) is added to the playlist which the player can call to retrieve the decryption key. The key is sent in the clear and the player decrypts the media objects in application space before providing them to the source buffer. This mode provides more security than [1] and [2] above, but at a lower cost than [3]

To date, the DASH ecosystem has lacked an equivalent to AES-128 encryption. To fill this gap and to provide a level of content protection between HTTPS-delivered token auth and DRM, we propose ClearKey Content Protection (CCP).

CCP leverages the ClearKey mode defined by Encrypted Media Extensions. It affords all the same protections as HLS  AES-128 with the additional benefit that the media segments are not decrypted by the player application, but rather by the Content Decryption Module (CDM) resident on the device. 

## Workflow
1. The manifest is populated with a ContentProtection element. The schemeIdUri's and default-KID shown MUST be used with values as shown. The `<dashif:laurl>` element should point at the relevant license server and it MUST use https.

    ```<ContentProtection schemeIdUri="urn:mpeg:dash:mp4protection:2011" value="cenc" cenc:default_KID="9eb4050d-e44b-4802-932e-27d75083e266" />'''
    <ContentProtection value="ClearKey1.0" schemeIdUri="urn:uuid:e2719d58-a985-b3c9-781a-b030af78d30e">
    <dashif:laurl>https://license_server_host.com/sessionID/license?token=example</dashif:laurl>
    </ContentProtection>
    ```
    

2. The player should remove the - from the the KID, convert the Hex to base64, strip out any == padding and then call the License server specified in the MPD laurl with the POST-based JSON request document specified in the https://www.w3.org/TR/encrypted-media/#clear-key-request-format 9.1.3 License Request Format:

    ```{"kids":["nrQFDeRLSAKTLifXUIPiZg"],"type":"temporary"}```

3. The server will respond with the JSON response specified in https://www.w3.org/TR/encrypted-media/#clear-key-license-format 9.1.4 License Format:

    ```{"keys":[{"kty":"oct","k":"FmY0xnWCPCNaSpRG-tUuTQ","kid":"nrQFDeRLSAKTLifXUIPiZg"}],"type":"temporary"}```

    which the player can then use to proceed with decryption via the CENC (Common Encryption) componentavailable on the device. 
    
    
 ## Examples
 
An example manifest is provided here:
https://dash-license.westus.cloudapp.azure.com/ClearKey_2160p/Manifest_ClearKey.mpd

A functioning license server at
https://dash-license.westus.cloudapp.azure.com/AcquireLicense

The dash.js player support server-side CCCP via this branch:
https://github.com/Dash-Industry-Forum/dash.js/tree/feature-clearkeyByServer

Issue tracker for dash.js integration:
https://github.com/Dash-Industry-Forum/dash.js/issues/3343
 

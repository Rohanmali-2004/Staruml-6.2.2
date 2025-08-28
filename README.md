# StarUML Full License Activation with compatible version 6.2.2


---

## 1. Install StarUML
Download the latest version of StarUML from the [official website](https://staruml.io/download).

---

## 2. Install `asar`
Next, install `asar`, a utility to manage `.asar` files. Open your terminal as an administrator and run the following command:

```bash
npm i asar -g
```
> [!IMPORTANT]
> Make sure to have the **LTS** version of Node.js installed to ensure compatibility and avoid errors when running `npm` commands. You can download the LTS version from [nodejs.org](https://nodejs.org/).

This will install `asar` globally.

---

## 3. Extract `app.asar`
To access the files needed to modify the license and export settings, extract the `app.asar` file.

Navigate to the StarUML directory. By default, it’s located at:

- **Windows**: `C:/Program Files/StarUML/resources`
- **MacOS**: `/Applications/StarUML.app/Contents/Resources`
- **Linux**: `/opt/staruml/resources`

You can use the `cd` command to navigate to your specific directory. For example:

```bash
cd "C:/Program Files/StarUML/resources"
```

Run the following command in your terminal as an Administrator (Git Bash, PowerShell, or CMD):

```bash
asar e app.asar app
```

This will extract the `app.asar` file into a folder called `app`.

---

## 4. Modify the License Manager

In the extracted files, navigate to the following path:

```
Program Files/StarUML/resources/app/src/engine/license-manager.js
```

Open the `license-manager.js` file in your preferred code editor and paste this modified license manager code here.

```js
const { EventEmitter } = require("events"); 
const fs = require("fs"); 
const path = require("path"); 
const crypto = require("crypto"); 
const UnregisteredDialog = require("../dialogs/unregistered-dialog"); 
const packageJSON = require("../../package.json");

const SK = "DF9B72CC966FBE3A46F99858C5AEE";

// Check License When File Save 
const LICENSE_CHECK_PROBABILITY = 0.3;

const PRO_DIAGRAM_TYPES = [
    "SysMLRequirementDiagram",
    "SysMLBlockDefinitionDiagram",
    "SysMLInternalBlockDiagram",
    "SysMLParametricDiagram",
    "BPMNDiagram",
    "WFWireframeDiagram",
    "AWSDiagram",
    "GCPDiagram",
];

var status = false; 
var licenseInfo = null;

/**
Set Registration Status
This function is out of LicenseManager class for the security reason
(To disable changing License status by API)
@private
@param {boolean} newStat
@return {string} 
*/ 
function setStatus(licenseManager, newStat) { 
    if (status !== newStat) { 
        status = newStat; 
        licenseManager.emit("statusChanged", status); 
    } 
}

/**
@private 
*/ 
class LicenseManager extends EventEmitter { 
    constructor() { 
        super(); 
        this.projectManager = null; 
    }

    isProDiagram(diagramType) { 
        return PRO_DIAGRAM_TYPES.includes(diagramType); 
    }

    /**
    Get Registration Status
    @return {string} 
    */ 
    getStatus() { 
        return status; 
    }

    /**
    Get License Infomation
    @return {Object} 
    */ 
    getLicenseInfo() { 
        return licenseInfo; 
    }

    findLicense() { 
        var licensePath = path.join(app.getUserPath(), "/license.key"); 
        if (!fs.existsSync(licensePath)) { 
            licensePath = path.join(app.getAppPath(), "../license.key"); 
        } 
        if (fs.existsSync(licensePath)) { 
            return licensePath; 
        } else { 
            return null; 
        } 
    }

    /**
    Check license validity
    @return {Promise} 
    */ 
    validate() { 
        return new Promise((resolve, reject) => { 
            try { 
                // Local check 
                var file = this.findLicense(); 
                if (!file) { 
                    reject("License key not found"); 
                } else { 
                    var data = fs.readFileSync(file, "utf8"); 
                    licenseInfo = JSON.parse(data); 
                    if (licenseInfo.product !== packageJSON.config.product_id) { 
                        app.toast.error(`License key is for old version (${licenseInfo.product})`); 
                        reject(`License key is not for ${packageJSON.config.product_id}`); 
                    } else { 
                        var base = SK + licenseInfo.name + SK + licenseInfo.product + "-" + licenseInfo.licenseType + SK + licenseInfo.quantity + SK + licenseInfo.timestamp + SK; 
                        var _key = crypto.createHash("sha1").update(base).digest("hex").toUpperCase(); 
                        if (_key !== licenseInfo.licenseKey) { 
                            reject("Invalid license key"); 
                        } else { 
                            // Server check 
                            $.post(app.config.validation_url, { licenseKey: licenseInfo.licenseKey, }) 
                                .done((data1) => { 
                                    resolve(data1); 
                                }) 
                                .fail((err) => { 
                                    if (err && err.status === 499) { 
                                        /* License key not exists */ 
                                        reject(err); 
                                    } else { 
                                        // If server is not available, assume that license key is valid 
                                        resolve(licenseInfo); 
                                    } 
                                }); 
                        } 
                    } 
                } 
            } catch (err) { 
                reject(err); 
            } 
        }); 
    }

    /**
    Return evaluation period status
    @private
    @return {number} Remaining days 
    */ 
    checkEvaluationPeriod() { 
        const file = path.join(window.app.getUserPath(), "lib.so"); 
        if (!fs.existsSync(file)) { 
            const timestamp = Date.now(); 
            fs.writeFileSync(file, timestamp.toString()); 
        } 
        try { 
            const timestamp = parseInt(fs.readFileSync(file, "utf8")); 
            const now = Date.now(); 
            const remains = 30 - Math.floor((now - timestamp) / (1000 * 60 * 60 * 24)); 
            return remains; 
        } catch (err) { 
            console.error(err); 
        } 
        return -1; // expired 
    }

    async checkLicenseValidity() { 
        // Instead of validating the license, always set status to true
        setStatus(this, true); 
    }

    /**
    Check the license key in server and store it as license.key file in local
    @param {string} licenseKey 
    */ 
    register(licenseKey) { 
        return new Promise((resolve, reject) => { 
            $.post(app.config.validation_url, { licenseKey: licenseKey }) 
                .done((data) => { 
                    if (data.product === packageJSON.config.product_id) { 
                        var file = path.join(app.getUserPath(), "/license.key"); 
                        fs.writeFileSync(file, JSON.stringify(data, 2)); 
                        licenseInfo = data; 
                        setStatus(this, true); 
                        resolve(data); 
                    } else { 
                        setStatus(this, false); 
                        reject("unmatched"); /* License is for old version */ 
                    } 
                }) 
                .fail((err) => { 
                    setStatus(this, false); 
                    if (err.status === 499) { 
                        /* License key not exists */ 
                        reject("invalid"); 
                    } else { 
                        reject(); 
                    } 
                }); 
        }); 
    }

    htmlReady() {}

    appReady() { 
        this.checkLicenseValidity(); 
    } 
}

module.exports = LicenseManager;
```

---
---

## 5. Repack `app.asar`

Once you have edited the necessary files, you need to repack the `app.asar` file. Navigate back to the `resources` directory and run the following command:

```bash
asar pack app app.asar
```

This will repack your modified `app` folder back into a `.asar` file.

---

## 6. Clean Up

After repacking the `app.asar`, you can safely remove the extracted `app` folder to clean up your directory.

### Remove the app folder:

- **For Windows:**
   ```bash
   rmdir /s /q app
   ```

- **For Linux or Mac:**
   ```bash
   rm -rf app
   ```

---

## 7. Launch StarUML

Now that everything is set up, launch StarUML by running the `StarUML.exe` file from your installation directory or through the desktop shortcut.

---

## Enjoy!

Congratulations! You now have StarUML fully licensed and can export diagrams in high resolution without watermarks.

> [!NOTE]
> This guide applies to StarUML version 6.2.2. Ensure you are using the correct version for compatibility.

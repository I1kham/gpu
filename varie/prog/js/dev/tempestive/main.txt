const isTEMPESTIVECLOUD = () => { return 1; };
const TEMPESTIVE_API = "nuboj6-prod-api.nuboj.eu";

class TempestiveCloud {

  /**
   * Init a websocket connection with Nuboj
   * @returns {WebSocket} The websocket instance
   */
  static init() {
    const params = new URL(location).searchParams;
    const token = params.get('token');
    const uniqueCode = params.get('uniqueCode');
    const wsAddr = `wss://${TEMPESTIVE_API}/rhea-websocket/${uniqueCode}/ws`;

    console.log('Tempestive:: trying to connect to ' + wsAddr);
    return new WebSocket(wsAddr, ['Authorization', `${token}`]);
  };

  static ws_onError(evt) {
    console.log("Tempestive::onWebSocketErr => ERROR: " + evt.data);
    const message = { type: 'tempestive-remote-ui', cause: 'ws-error', error: evt.data };
    window.parent.postMessage(message, "*");
    setTimeout(function () { TempestiveCloud.relaod() }, 2000);
    setTimeout(function () { reject(-1); }, 5000);
  }

  static ws_onTimeout(msg) {
    console.log("Tempestive::onWebSocketTimeout => " + msg);
    const message = { type: 'tempestive-remote-ui', cause: 'ws-timeout', error: msg };
    window.parent.postMessage(message, "*");
  }

  static relaod() {
    const newURL = this.updatePageURL(new URL(location), Rhea_session_getOrDefault("lang", "GB"), "", "")
    window.location = newURL.toString();
  }

  /**
   * Updates the URL to match new parameters
   *
   * @param {URL} previousURL - The URL to be updated
   * @param {string} langISO - The language in ISO format, es: EN, DE, IT...
   * @param {string} profileID - The user profile ID
   * @param {string} page - The page to be redirected to
   *
   * @returns {URL} The updated URL
   */
  static updatePageURL(previousURL, langISO, profileID, page) {
    const pageName = `index_${langISO}.html`;
    const newURL = new URL(pageName, previousURL);

    const params = new URL(previousURL).searchParams;
    const token = params.get('token');
    const uniqueCode = params.get('uniqueCode');
    const id = params.get('id');
    const mode = params.get('mode');

    // Update parameters
    if (profileID !== "") {
      newURL.searchParams.set('userP', profileID);
    } else {
      newURL.searchParams.delete('userP');
    }

    if (page !== "") {
      newURL.searchParams.set('page', page);
    } else {
      newURL.searchParams.delete('page');
    }

    if (uniqueCode !== "") {
      newURL.searchParams.set('uniqueCode', uniqueCode);
    } else {
      newURL.searchParams.delete('uniqueCode');
    }

    if (token !== "") {
      newURL.searchParams.set('token', token);
    } else {
      newURL.searchParams.delete('token');
    }

    if (id !== "") {
      newURL.searchParams.set('id', id);
    } else {
      newURL.searchParams.delete('id');
    }

    if (mode && mode !== "") {
      newURL.searchParams.set('mode', mode);
    } else {
      newURL.searchParams.delete('mode');
    }

    return newURL;

  }

  /**
   * Load the machine's configuration file     
   */
  static async load() {
    try {
      let config = await this.getConfig();
      let buffer = config.arrayBuffer();
      let bufferObj = {
        fileSize: buffer.byteLength,
        fileBuffer: new Uint8Array(buffer),
      };

      console.log("[Tempestive Cloud] - Config file loaded")
      DA3_load_onEnd(da3, 0, bufferObj);

    } catch (error) {
      console.error(error);
      DA3_load_onEnd(da3, -1, null)
    }
  }

  /**
   * Save the config file to Nuboj and quit
   */
  static async quit() {
    console.log("Tempestive::saveAndQuit => Closing remote sessions...");
    const message = { type: 'tempestive-remote-ui', cause: 'ui-quit' };
    window.parent.postMessage(message, "*");
    
    this.onQuitFinished()
  }

  /**
   * Show messages when save of config file finishes
   * @param {string} error - The error if present
   */
  static onQuitFinished(error = "") {
    pleaseWait_show();
    pleaseWait_rotella_hide();
    if (errorMessage == "")
      pleaseWait_freeText_setText("The machine will reboot shortly.<br>You can leave this page now.");
    else
      pleaseWait_freeText_setText("There was an errore while saving the configuration file.<br>You can leave this page now.");
    pleaseWait_freeText_show();
  }

  /**
   * Get file config of machine
   *
   * @returns {object} The json config of machine
   */
  static async getConfig() {
    const params = new URL(location).searchParams;
    const token = params.get('token');
    const sensorSetId = params.get('id');

    const requestUrl = `https://${TEMPESTIVE_API}/rhea/v1/MachineConfigurationFile/${sensorSetId}`;

    //get file
    const config = await fetch(requestUrl, {
      method: "get",
      headers: {
        Authorization: `Bearer ${token}`,
        "Content-Type": "application/json"
      }
    });

    return config;
  }
}
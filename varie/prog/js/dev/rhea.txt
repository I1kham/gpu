//Se si richiede un linguaggio che non è supportato (ie: mancano i file), allora si carica il RHEA_DEFAULT_FALLOFF_LANGUAGE
var 	RHEA_DEFAULT_FALLOFF_LANGUAGE = "GB";
var		RHEA_NUM_MAX_SELECTIONS = 64;

//costanti di comodo da passare come parametro alla fn requestGPUEvent() e/o ricevuti come eventi dalla GPU
var RHEA_EVENT_SELECTION_AVAILABILITY_UPDATED = 97; 	//'a'
var	RHEA_EVENT_SELECTION_PRICES_UPDATED = 98;			//'b'
var	RHEA_EVENT_CREDIT_UPDATED = 99;						//'c'
var	RHEA_EVENT_CPU_MESSAGE = 100;						//'d'
var	RHEA_EVENT_SELECTION_REQ_STATUS = 101;				//'e'
var	RHEA_EVENT_START_SELECTION = 102;					//'f'
var	RHEA_EVENT_STOP_SELECTION = 103;					//'g'
var	RHEA_EVENT_CPU_STATUS = 104;						//'h'
var	RHEA_EVENT_ANSWER_TO_IDCODE_REQUEST = 105;			//'i'
var	RHEA_EVENT_SEND_BUTTON = 115;						//'s'
var	RHEA_EVENT_SEND_PARTIAL_DA3 = 116;					//'t'
var	RHEA_EVENT_READ_DATA_AUDIT = 108;					//'l'
var	RHEA_EVENT_CPU_ACTIVATE_BUZZER = 71;				//'G'
var	RHEA_EVENT_SET_GUI_URL = 77;						//'M'



//info sulla versione attuale del codice (viene comunicata a GPU in fase di registrazione)
var RHEA_CLIENT_INFO__API_VERSION = 0x01;
var RHEA_CLIENT_INFO__APP_TYPE = 0x01;
var RHEA_CLIENT_INFO__UNUSED2 = 0x00;
var RHEA_CLIENT_INFO__UNUSED3 = 0x00;


/******************************************************
 * clearSessionData
 * 
 * Utile per pulire le informazioni di sessione. Di solito usata allo startup della GUI
 */
Rhea_clearSessionData = function (defaultLanguage)
{
	window.name = "";
	Rhea_session_setValue ("lang", RHEA_DEFAULT_FALLOFF_LANGUAGE);
	Rhea_session_setValue("credit", "0");
	Rhea_session_setValue("debug", 0);
	Rhea_session_setValue("debug_console", "");

	//selection session clear
	for (var i=1; i<=RHEA_NUM_MAX_SELECTIONS; i++)
		Rhea_session_clearObject("selInfo" +i);
}


/*********************************************************

 * Rhea
 *
 * costruttore
 */
function Rhea()
{
	//lingua corrente
	if (Rhea_session_getValue ("lang") === undefined)
		Rhea_session_setValue ("lang", RHEA_DEFAULT_FALLOFF_LANGUAGE);
	
	/*Recupero il mio idCode che mi identifica con GPUServer
	 *Non posso contare su session.storage() e nemmeno su i cookie perchè chrome e molti altri browser non supportano
	 *l'uso di questi tool quando si caricano file in locale (ie: file:/// invece che http://).
	 *Ho bisogno di un posto che posso usare per memorizzare il mio idCode e che sia diverso per ogni sessione. Se apro una seconda
	 *finestra nel browser, questa 2nda finestra deve poter ottenere un nuovo idCode senza interferire con quello della prima finestra.
	 *Ho scoperto che la proprietà window.name è associata ad ogni tab del browser, quindi uso quella per memorizzare l'idCode di ogni
	 *sessione aperta*/
	//if (this.idCode_0 == 0)
	if (window.name == "")
	{
		this.idCode_0 = 0;
		this.idCode_1 = 0;
		this.idCode_2 = 0;
		this.idCode_3 = 0;
	}
	else
	{
		this.idCode_0 = parseInt (window.name.substr(4,3));
		this.idCode_1 = parseInt (window.name.substr(8,3));
		this.idCode_2 = parseInt (window.name.substr(12,3));
		this.idCode_3 = parseInt (window.name.substr(16,3));
	}
	
	//ajax    
	this.nextAjaxRequestID = 1;
	this.ajaxReceivedAnswerQ = [];
	var NUM_MAX_CONCURRENT_AJAX_REQUEST = 16;	//num max di richieste "ajax" contemporanee
	for (var i=0; i<NUM_MAX_CONCURRENT_AJAX_REQUEST; i++)
		this.ajaxReceivedAnswerQ[i] = null;
	

	//Selezioni: array di sessione.
	//Alcune informazioni sullo stato delle selezioni sono permanenti, nel senso che sono memorizzate in "session".
	//L'aggiornamento di queste informazioni viene gestito in autonomia e le pagine possono sempre in ogni momento richiedere lo stato delle selezioni
	//utilizzando la fn selection_getBySelNumber()
	//Per sapere quali informazioni sono disponibili per ogni selezione, cerca la fn selection_createEmpty()
	this.selection_sessionRestore();
	
	
	//Variabile di sessione: credito attuale
	//Viene gestita in autonomia, in modo che this.credit rifletta sempre l'attuale credito disponibile
	if (!Rhea_session_getValue("credit"))
	{
		this.credit = "0";
		Rhea_session_setValue("credit", "0");
	}
	else
		this.credit = Rhea_session_getValue("credit");
	
	//file transfer
	this.nFileTransfer = 0;
	this.fileTransfer = [];
	
	this.partialDA3AckRcvd = -1;
}




/******************************************************
 * webSocket_connect
 *
 * 	Ritorna una promise che viene risolta quando la connessione è stabilita.
 *	In caso di errori, viene fatto automaticamente un retry della connessione. 
 *	Continua a fare retry fino a che la connessione è stabilita.
 */
Rhea.prototype.webSocket_connect = function()
{
	var me = this;
	return new Promise( function(resolve, reject) 
	{
		if(isTEMPESTIVECLOUD())
		{
			me.websocket = TempestiveCloud.init();
		}
		else 
		{
			var ip = "ws://127.0.0.1:2280/";
			//var ip = "ws://127.0.0.1:8200/";
			rheaLog("Rhea:: trying to connect to " +ip +"...");
			me.websocket = new WebSocket(ip, "binary");
		}
		
		
		me.websocket.onopen = function(evt) 
		{ 
			rheaLog("Rhea::webSocket connected...");
			
			if (me.idCode_0 == 0)
			{
				me.webSocket_requestIDCodeAfterConnection();
				
				//attendo la risposta
				var timeoutMs = 10000;
				var waitIDCode = function()	
				{
					if (me.idCode_3 != 0)
					{
						me.webSocket_identifyAfterConnection();
						resolve(1);
					}
					else if ((timeoutMs -= 100) < 0)
					{
						if(isTEMPESTIVECLOUD())
						{
							TempestiveCloud.ws_onTimeout("timed out 'waiting idcode");
						}						
						reject ("timed out 'waiting idcode");
						reject(0);
					}
					else
						setTimeout(waitIDCode, 100);
				}

				setTimeout(waitIDCode, 100)				
			}
			else
			{
				me.webSocket_identifyAfterConnection();
				resolve(1); 
			}
		};
		me.websocket.onclose = 		function(evt) { me.webSocket_onClose(evt); };
		me.websocket.onmessage = 	function(evt) { me.webSocket_onRcv(evt) };
		
		if(isTEMPESTIVECLOUD())
		{
			me.websocket.onerror = TempestiveCloud.ws_onError
		} 		
		else
		{
			me.websocket.onerror = function(evt) { rheaLog("Rhea::onWebSocketErr => ERROR: " + evt.data); setTimeout( function() {window.location="startup.html";}, 2000); };
		}
		
		setTimeout( function() {reject(-1)}, 5000);
	});
}



/*********************************************************
 * Websocket event handling
 *
 */
Rhea.prototype.webSocket_onClose = function (evt)
{
	rheaLog("Rhea::webSocket_onClose =>  Disconnected");
	this.websocket.close();
}

Rhea.prototype.priv_findFileTransferByAppUID = function (appTransfUID)
{
	for (var i=0; i<this.nFileTransfer; i++)
	{
		if (this.fileTransfer[i] == null)
			continue;
		if (this.fileTransfer[i].appTransfUID == appTransfUID)
			return i;
	}
	return -1;
}


function DEBUG_UTILS_logUint8Array (data)
{
	var out = "size=" +data.length +", ";
	for (var i=0; i<data.length; i++)		
		out += data[i].toString() +" ";
	console.log (out);
}

Rhea.prototype.webSocket_onRcv = function (evt)
{
	//rheaLog("RESPONSE: [len=" +evt.data.size +"] " + evt.data);
	var me = this;
	
	var fileReader = new FileReader();
	fileReader.readAsArrayBuffer(evt.data);
	fileReader.onload = function(event) 
	{
		var data = new Uint8Array(fileReader.result);
		var n = me.priv_parseRCV (data);
		if (n < data.length)
		{
			var totalSize = data.length;
//console.log ("n="+n +", totalSize="+totalSize);
//DEBUG_UTILS_logUint8Array (data);

			while (n < totalSize)
			{
				var newSize = totalSize-n;
				var data2 = new Uint8Array(totalSize-n);
				for (var i=0; i<newSize; i++)
					data2[i] = data[i+n];
//DEBUG_UTILS_logUint8Array (data2);
				n += me.priv_parseRCV (data2);
			}
		}
		
		//rheaLog ("Rhea::RCV [len=" +data.length +"] [" +utf8ArrayToStr(data) +"]");
		
		
	}
}

/***********************************************************
	[data] e' un Uint8Array
	ritorna il numero di byte di data che sono stati utilizzati
*/
Rhea.prototype.priv_parseRCV = function (data)
{
	//vediamo se è una risposta "ajax" (il buffer deve iniziare con #AJ.ja dove al posto di "." c'è la requestID
	if (data.length > 6)
	{
		//# A J [reqID] j a [stringa_utf8]
		//# A J [reqID] j a [0x01] [0x02] [0x03] [0x04] [0x05] [stringa_utf16]
		if (data[0] == 35 && data[1] == 65 && data[2] == 74 && data[4] == 106 && data[5] == 97)
		{
			var reqID = parseInt(data[3]);

			for (var i=0; i<this.ajaxReceivedAnswerQ.length; i++)
			{
				if (this.ajaxReceivedAnswerQ[i] == null)
					continue;
				if (this.ajaxReceivedAnswerQ[i].requestID == reqID)
				{
					//console.log ("AJAX rcv for reqID=" +reqID);
					var isUTF16 = 0;
					if (data.length > 11)
					{
						if (data[6]==0x01 && data[7]==0x02 && data[8]==0x03 && data[9]==0x04 && data[10]==0x05)
							isUTF16 = 1;
					}


					try
					{
						var offset = 6;
						if (isUTF16)
						{
							offset = 11;
							this.ajaxReceivedAnswerQ[i].rcv = utf16ArrayToStr(data.subarray(offset));
						}
						else
							this.ajaxReceivedAnswerQ[i].rcv = utf8ArrayToStr(data.subarray(offset));
						
						
						//in rare circostanze, il msg ajax e' subito seguito ad un messaggio di evento e il pacchetto viene spedito
						//come un tutt'uno
						var str_length = this.ajaxReceivedAnswerQ[i].rcv.indexOf("#eVn");
						if (-1 !== str_length)
						{
							//console.error ("beccato, offset=" +offset); console.log ("src="+this.ajaxReceivedAnswerQ[i].rcv);
							this.ajaxReceivedAnswerQ[i].rcv = this.ajaxReceivedAnswerQ[i].rcv.substr (0, str_length);
							//console.log ("used="+this.ajaxReceivedAnswerQ[i].rcv);
							return offset + str_length;
						}
						
						str_length = this.ajaxReceivedAnswerQ[i].rcv.indexOf("#AJ");
						if (-1 !== str_length)
						{
							//console.error ("beccato2, offset=" +offset); console.log ("src="+this.ajaxReceivedAnswerQ[i].rcv);
							this.ajaxReceivedAnswerQ[i].rcv = this.ajaxReceivedAnswerQ[i].rcv.substr (0, str_length);
							//console.log ("used="+this.ajaxReceivedAnswerQ[i].rcv);
							return offset + str_length;
						}
						
						str_length = this.ajaxReceivedAnswerQ[i].rcv.indexOf("#fTr");
						if (-1 !== str_length)
						{
							//console.error ("beccato3, offset=" +offset); console.log ("src="+this.ajaxReceivedAnswerQ[i].rcv);
							this.ajaxReceivedAnswerQ[i].rcv = this.ajaxReceivedAnswerQ[i].rcv.substr (0, str_length);
							//console.log ("used="+this.ajaxReceivedAnswerQ[i].rcv);
							return offset + str_length;
						}						
						
						
					}
					catch(err)
					{
						console.error ("error=" +err);
					}
					
					return data.length;
				}
			}
			return data.length;
		}
	}
	
	//vediamo se è un evento inviato dalla GPU (il buffer deve iniziare con #eVn. dove al posto di "." c'è un byte che indica il tipo di evento
	if (data.length >= 5)
	{
		//# e V n [eventTypeID] [eventSeqNum] [payloadLen MSB_LSB] [payload]
		if (data[0] == 35 && data[1] == 101 && data[2] == 86 && data[3] == 110)
		{
			var eventTypeID = parseInt(data[4]);
			var eventSeqNum = parseInt(data[5]);
			var payloadLen = parseInt(256 * data[6]) + parseInt(data[7]);
			
			switch (eventTypeID)
			{
			case RHEA_EVENT_ANSWER_TO_IDCODE_REQUEST:
				{
					if (this.idCode_0 == 0)
					{
						//var cpuBridgeVersion = parseInt(data[8]);
						//var socketBridgeVersion = parseInt(data[9]);
						this.idCode_0 = parseInt(data[10]);
						this.idCode_1 = parseInt(data[11]);
						this.idCode_2 = parseInt(data[12]);
						this.idCode_3 = parseInt(data[13]);
						
						var s1 = this.idCode_0.toString(); while (s1.length<3) s1="0"+s1;
						var s2 = this.idCode_1.toString(); while (s2.length<3) s2="0"+s2;
						var s3 = this.idCode_2.toString(); while (s3.length<3) s3="0"+s3;
						var s4 = this.idCode_3.toString(); while (s4.length<3) s4="0"+s4;
						var s = "rhea" +s1 +"-" +s2 +"-" +s3 +"-" +s4;
						window.name = s;
					}
				}
				break;
				
			case RHEA_EVENT_SELECTION_AVAILABILITY_UPDATED:
				{
					rheaLog ("RHEA_EVENT_SELECTION_AVAILABILITY_UPDATED:");
					var nSel = parseInt(data[8]);
					for (var i=1; i<=nSel; i++)
						this.selection_getBySelNumber(i).enabled = parseInt(data[8+i]);
					this.selection_sessionStore();
					this.onEvent_selectionAvailabilityUpdated();
				}
				break;
				
			case RHEA_EVENT_SELECTION_PRICES_UPDATED:
				{
					rheaLog ("RHEA_EVENT_SELECTION_PRICES_UPDATED:");
					var prices = utf8ArrayToStr(data.subarray(8)).split("§")
					for (var i=1; i<=prices.length; i++)
						this.selection_getBySelNumber(i).price = prices[i-1];
					this.selection_sessionStore();
					this.onEvent_selectionPricesUpdated();
				}
				break;
				
			case RHEA_EVENT_CREDIT_UPDATED:
				{
					rheaLog ("RHEA_EVENT_CREDIT_UPDATED:");
					this.credit = utf8ArrayToStr(data.subarray(8));
					Rhea_session_setValue("credit", this.credit);
					this.onEvent_creditUpdated();
				}
				break;
				
			case RHEA_EVENT_CPU_MESSAGE:
				{
					var importanceLevel = data[8];
					var lenInBytes = data[9] * 256 + data[10];
					var msg = utf16ArrayToStr(data.subarray(11));
					//rheaLog ("RHEA_EVENT_CPU_MESSAGE: [" +importanceLevel +"] " +msg);
					if (msg != "")
						this.onEvent_cpuMessage(msg, importanceLevel);
				}
				break;					
			
			case RHEA_EVENT_SELECTION_REQ_STATUS:
				{
					var status = parseInt(data[8]);
					rheaLog ("RHEA_EVENT_SELECTION_REQ_STATUS [" +status +"]");
					this.onEvent_selectionReqStatus(status);
				}
				break;
					
			case RHEA_EVENT_CPU_STATUS:
				var statusID = parseInt(data[8]);
				//var errorCode = data[9];
				//var errorType = data[10];
				//console.log ("Rhea::RCV RHEA_EVENT_CPU_STATUS = " +statusID +", datalen=" +data.length);

				var flag16 = parseInt(data[11] * 256 + data[12]);
				var statusStr = "";
				switch (statusID)
				{
					case 2: statusStr ="READY"; break;
					case 3: statusStr ="DRINK PREP"; break;
					case 4: statusStr ="PROGR"; break;
					case 5: statusStr ="INI CHECK"; break;
					case 6: statusStr ="ERROR"; break;
					case 7: statusStr ="RINSING"; break;
					case 8: statusStr ="AUTO WASHING"; break;
					case 10: statusStr ="WAIT TEMP"; break;
					case 13: statusStr ="DISINSTALLATION"; break;
					case 15: statusStr ="END DISINSTALLATION"; break;
					case 20: statusStr ="BREWER CLEANING"; break;
					case 21: statusStr ="TEST SEL"; break;
					case 22: statusStr ="TEST MODEM"; break;
					case 23: statusStr ="CLEANING MILKER"; break; //VENTURI
					case 24: statusStr ="CLEANING MILKER"; break; //INDUX
					case 26: statusStr ="DESCALING"; break; //INDUX
					case 30: statusStr ="MILKER DESCALING"; break;	//DESCALING CAPPUCCINATORE INDUX
					case 31: statusStr ="SYRUP CLEANING"; break; 	//Modulo sciroppi
					case 101: statusStr ="COM_ERROR"; break;
					case 102: statusStr ="GRINDER OPENING"; break;
					
					case 103: statusStr ="COMPATIBILITY CHECK"; break;
					case 104: statusStr ="CPU_NOT_SUPPORTED"; break;
					case 105: statusStr ="DA3_SYNC"; break;
					case 106: statusStr ="GRINDER SPEED TEST"; break;
				}
				
				//console.log ("rhea.js => RHEA_EVENT_CPU_STATUS [" +statusID +"] [" +statusStr +"]");
				this.onEvent_cpuStatus(statusID, statusStr, flag16);
				break;
				
			case RHEA_EVENT_SEND_PARTIAL_DA3:
				//notifica da parte della SMU che ci dice che un blocco di DA3 è stato scritto
				this.partialDA3AckRcvd = parseInt(data[8])*256 + parseInt(data[9]);
				//console.log ("RHEA_EVENT_SEND_PARTIAL_DA3 [" +this.partialDA3AckRcvd +"]");
				break;
				
			case RHEA_EVENT_READ_DATA_AUDIT:
				fileID = data[8] * 256 + data[9];
				kbSoFar = data[10] * 256 + data[11];
				status = data[12];
				this.onEvent_readDataAudit(status, kbSoFar, fileID)
				break;
				
			case RHEA_EVENT_SET_GUI_URL:
				var newURL = utf8ArrayToStr(data.subarray(8));
				console.log ("redirect to " +newURL);
				window.location = newURL;
				break;
			}
			
			return (8 + payloadLen);
		}
	}


	//vediamo se è un messaggio relativo al protocollo di filetransfer  (il buffer deve iniziare con #fTr.. dove al posto di ".." c'è u16 che indica il size dei dati che seguono
	if (data.length >= 9)
	{
		//# f T r [payloadLen MSB_LSB] [payload...] [ck16]
		if (data[0] == 35 && data[1] == 102 && data[2] == 84 && data[3] == 114)
		{
			var sizeOfWhat = parseInt(data[4])*256 + parseInt(data[5]);
			var opcode = parseInt(data[6]);
			
			var totalMsgLen = 8 + sizeOfWhat;
			switch (opcode)
			{
				case 2:
					var sData0x02 = new Object();
					sData0x02.reason_refused = parseInt(data[7]);
					sData0x02.packetSizeInBytes = parseInt(data[8])*256 + parseInt(data[9]);
					sData0x02.smuTransfUID = (parseInt(data[10])<<24) | (parseInt(data[11])<<16) | (parseInt(data[12])<<8) |parseInt(data[13]);
					sData0x02.appTransfUID = (parseInt(data[14])<<24) | (parseInt(data[15])<<16) | (parseInt(data[16])<<8) |parseInt(data[17]);
					sData0x02.numPacketInAChunk = parseInt(data[18]);
					
					for (var i=0; i<this.nFileTransfer; i++)
					{
						if (this.fileTransfer[i].appTransfUID == sData0x02.appTransfUID)
						{
							if (!this.fileTransfer[i].priv_on0x02 (sData0x02))
								this.fileTransfer[i] = null;
							return totalMsgLen;
						}
					}
					rheaLog ("ERR, file transfer not found. sData0x02.appTransfUID[" +ssData0x02.appTransfUID +"] sData0x02.smuTransfUID[" +sData0x02.smuTransfUID +"]");
					break;
					
				case 4:
					var sData0x04 = new Object();
					sData0x04.appTransfUID = (parseInt(data[7])<<24) | (parseInt(data[8])<<16) | (parseInt(data[9])<<8) | parseInt(data[10]);
					sData0x04.packetNumAccepted = (parseInt(data[11])<<24) | (parseInt(data[12])<<16) | (parseInt(data[13])<<8) | parseInt(data[14]);
					for (var i=0; i<this.nFileTransfer; i++)
					{
						if (this.fileTransfer[i].appTransfUID == sData0x04.appTransfUID)
						{
							if (!this.fileTransfer[i].priv_on0x04 (sData0x04))
								this.fileTransfer[i] = null;
							return totalMsgLen;
						}
					}
					
					rheaLog ("ERR, file transfer not found. sData0x04.appTransfUID[" +sData0x04.appTransfUID +"] sData0x04.packetNumAccepted[" +sData0x04.packetNumAccepted +"]");
					break;	

				case 0x52:
					var sData0x52 = new Object();
					sData0x52.reason_refused = parseInt(data[7]);
					sData0x52.packetSizeInBytes = parseInt(data[8])*256 + parseInt(data[9]);
					sData0x52.smuTransfUID = (parseInt(data[10])<<24) | (parseInt(data[11])<<16) | (parseInt(data[12])<<8) |parseInt(data[13]);
					sData0x52.appTransfUID = (parseInt(data[14])<<24) | (parseInt(data[15])<<16) | (parseInt(data[16])<<8) |parseInt(data[17]);
					sData0x52.fileSize = (parseInt(data[18])<<24) | (parseInt(data[19])<<16) | (parseInt(data[20])<<8) |parseInt(data[21]);
					sData0x52.numPacketInAChunk = parseInt(data[22]);
					
					var i = this.priv_findFileTransferByAppUID (sData0x52.appTransfUID);
					if (i>=0)
					{
						if (!this.fileTransfer[i].priv_on0x52 (sData0x52))
							this.fileTransfer[i] = null;
						return totalMsgLen;
					}
					rheaLog ("ERR, file transfer not found. sData0x52.appTransfUID[" +sData0x52.appTransfUID +"] sData0x02.smuTransfUID[" +sData0x52.smuTransfUID +"]");
					break;
					
				case 0x53:
					var sData0x53 = new Object();
					sData0x53.appTransfUID = (parseInt(data[7])<<24) | (parseInt(data[8])<<16) | (parseInt(data[9])<<8) | parseInt(data[10]);
					sData0x53.packetNumReceived = (parseInt(data[11])<<24) | (parseInt(data[12])<<16) | (parseInt(data[13])<<8) | parseInt(data[14]);
					sData0x53.chunkSeq = parseInt(data[15]);
					
					var i = this.priv_findFileTransferByAppUID (sData0x53.appTransfUID);
					if (i>=0)
					{
						var n = this.fileTransfer[i].priv_on0x53 (sData0x53);
						if (n > 0)
						{
							for (var i2=0; i2<n; i2++)
								this.fileTransfer[i].fileBuffer[this.fileTransfer[i].fileBufferCT++] = data[16+i2];
						}
						else if (n < 0)
						{
							n = -n;
							for (var i2=0; i2<n; i2++)
								this.fileTransfer[i].fileBuffer[this.fileTransfer[i].fileBufferCT++] = data[16+i2];
							this.fileTransfer[i].callback_onEnd (this.fileTransfer[i].userValue, 0, this.fileTransfer[i]);
							this.fileTransfer[i] = null;									
						}
						return totalMsgLen;
					}			

					rheaLog ("ERR, file transfer not found. sData0x53.appTransfUID[" +sData0x53.appTransfUID +"] sData0x53.packetNumReceived[" +sData0x53.packetNumReceived +"]");
					break;						
					
				default:						
					rheaLog ("#fTr, opcode[" +opcode +"]");
					break;
			}
			
			return totalMsgLen;
		}
	}
	
	//è arrivato qualcosa di strano
	rheaLog ("Rhea::RCV [len=" +data.length +"] [" +utf8ArrayToStr(data) +"]");
	return data.length;	
}

/***********************************************************
 * sendGPUCommand
 *
 *  invia un comando alla GPU.
 *	Ritorna una promise se [bReturnAPromise]==1
 *	Un comando è composto da un commandChar (ovvero un carattere ASCII) e un buffer di byte a seguire
 */
Rhea.prototype.sendGPUCommand = function (commandChar, bufferArrayIN, bReturnAPromise, promiseTimeoutMSec)
{
	var requestID = 255;
	var iReq;
	
	if (bReturnAPromise)
	{
		//requestID deve essere un byte, e deve essere maggiore di 0.
		//Ho messo il limite a 200, ma è un valore arbitrario, non ha un senso specifico. Serve solo ad evitare di eccedere
		//la capienza di un byte. Potevo usare 255 come limite, ma non si sa mai che in futuro si voglia riservare specifici
		//requestID per specifiche richieste
		var requestID = this.nextAjaxRequestID++;
		if (this.nextAjaxRequestID > 200)	
			this.nextAjaxRequestID = 1;

		
		for (iReq=0; iReq<this.ajaxReceivedAnswerQ.length; iReq++)
		{
			if (this.ajaxReceivedAnswerQ[iReq] == null)
			{
				this.ajaxReceivedAnswerQ[iReq] = new RheaAjaxAnswer (requestID);
				break;
			}
		}
			
		if (iReq >= this.ajaxReceivedAnswerQ.length)
		{
			rheaLog ("Rhea::GPUCommand => too many request");
			return new Promise ( function(resolve, reject) { reject("Rhea::GPUCommand => too many request")} );
		}
	} 	

	//prepara il buffer da inviare
	var bufferLen = 0;
	if (null != bufferArrayIN)
		bufferLen = bufferArrayIN.length;
	var bytesNeeded = 	1 +				//# 
						1 +				//char del comando
						1 +				//requestID
						2 +				//lunghezza di [bufferArrayIN] espressa come byte alto, byte basso
						bufferLen +		//[bufferArrayIN]
						2;				//cheksum espressa come byte alto, byte basso
	
	var buffer = new Uint8Array(bytesNeeded);
		var t = 0;
		buffer[t++] = 35; //'#'
		buffer[t++] = commandChar.charCodeAt(0);
		buffer[t++] = requestID;
		
		buffer[t++] = ((bufferLen & 0xFF00) >> 8);
		buffer[t++] = (bufferLen & 0x00FF);
		for (var i=0; i<bufferLen; i++)
			buffer[t++] = bufferArrayIN[i];

		//checksum
		var ck = 0;
		for (var i=0; i<t; i++)
			ck += buffer[i];

		buffer[t++] = ((ck & 0xFF00) >> 8);
		buffer[t++] = (ck & 0x00FF);

	//invia
	this.websocket.send(buffer);
	
	if (bReturnAPromise == 0)
		return;


	var me = this;
	return new Promise( function(resolve, reject) 
					{
						var timeoutMs = promiseTimeoutMSec;
						if (isTEMPESTIVECLOUD())
							timeoutMs += 8000; // additional timeout to account for network latency

						var check = function()	{
												//console.warn("checking[" +iReq +"]")
												if (me.ajaxReceivedAnswerQ[iReq].rcv != null)
												{
													//console.log ("ireq="+iReq);
													resolve(me.ajaxReceivedAnswerQ[iReq].rcv);
													me.ajaxReceivedAnswerQ[iReq] = null;
												}
												else if ((timeoutMs -= 25) < 0)
												{
													reject ("timed out [reqID:" +me.ajaxReceivedAnswerQ[iReq].requestID +"][pos:" +iReq +"]");
													me.ajaxReceivedAnswerQ[iReq] = null;
												}
												else
													setTimeout(check, 25);
											}

						setTimeout(check, 25);
					});			
};

/***********************************************************
 * ajax
 *
 * 	Simula il meccanismo di ajax utlizzando il websocket come layer di trasporto.
 *	Ritorna una promise.
 *	Esempio di una richiesta con commandString="time" e vari parametri accessori
		rhea.ajax ("time", 
						{
							"name" : "pippo",
							"number" : i
						})
			.then( function(result) 
			{
				var obj = JSON.parse(result);
				alert (obj.numero);
			})
			.catch( function(result)
			{
				alert ("AJAX::error => " +result);
			});
 */
Rhea.prototype.ajax = function(commandString, plainJSObject)
{
	return this.ajaxWithCustomTimeout(commandString, plainJSObject, 4000);
};
Rhea.prototype.ajaxWithCustomTimeout = function(commandString, plainJSObject, timeoutMSec)
{
	if (plainJSObject=="")
		plainJSObject="{}";
	var jo = JSON.stringify(plainJSObject);
	//jo  = jo.replace(/[\u007F-\uFFFF]/g, function(chr) { return "\\u" + ("0000" + chr.charCodeAt(0).toString(16)).substr(-4)})

	var bytesNeeded = 	1 							//length di [commandString]
						+ commandString.length		//[commandString]
						+ 2							//length di [plainJSObject]
						+ jo.length					//[plainJSObject]

	var buffer = new Uint8Array(bytesNeeded);
		var t = 0;
		buffer[t++] = (commandString.length & 0x00FF);
		for (var i=0; i<commandString.length; i++)
			buffer[t++] = commandString.charCodeAt(i);

		buffer[t++] = ((jo.length & 0xFF00) >> 8);
		buffer[t++] = (jo.length & 0x00FF);
		for (var i=0; i<jo.length; i++)
			buffer[t++] = jo.charCodeAt(i);


	//console.log ("ajax [" +commandString +"] [" +timeoutMSec +"ms]");
	return this.sendGPUCommand ("A", buffer, 1, timeoutMSec);
};


/*********************************************************
 * requestGPUEvent
 *
 *	invia una richiesta alla GPU in modo che questa, a sua volta, invii un messaggio
 *	per scatenare l'evento indicato
 *	
 * 	Il parametro eventTypeID può valere una delle const che trovi ad inizio file e il cui nome inizia con RHEA_EVENT_
 */
Rhea.prototype.requestGPUEvent = function (eventTypeID)
{
	var buffer = new Uint8Array(1);
	buffer[0] = eventTypeID;
	this.sendGPUCommand ("E", buffer, 0, 0);
}

/*******************************************************************
 * richiede di fare un lavaggio
 */
Rhea.prototype.requestGPUCleaning = function (cleanType)
{
	this.sendCPUProgrammingCmd(2,cleanType,0,0,0);
}

/*******************************************************************
 * richiede un comando di programmazione con i suoi parametri 
 */
Rhea.prototype.sendCPUProgrammingCmd = function (cmd, param1, param2, param3, param4)
{
	var buffer = new Uint8Array(6);
	buffer[0] = 113; //'q';
	buffer[1] = parseInt(cmd);
	buffer[2] = parseInt(param1);
	buffer[3] = parseInt(param2);
	buffer[4] = parseInt(param3);
	buffer[5] = parseInt(param4);
	this.sendGPUCommand ("E", buffer, 0, 0);
}

Rhea.prototype.sendButtonPress = function(iBtnNumber)
{
	var buffer = new Uint8Array(2);
	buffer[0] = RHEA_EVENT_SEND_BUTTON;
	buffer[1] = parseInt(iBtnNumber);
	this.sendGPUCommand ("E", buffer, 0, 0);
}

Rhea.prototype.sendStartDisintallation = function()
{
	var buffer = new Uint8Array(1);
	buffer[0] = 65;
	this.sendGPUCommand ("E", buffer, 0, 0);
}

Rhea.prototype.sendSyncDA3 = function()
{
	var buffer = new Uint8Array(1);
	buffer[0] = 67; //C
	this.sendGPUCommand ("E", buffer, 0, 0);
}

Rhea.prototype.sendGetCPUStatus = function()
{
	var buffer = new Uint8Array(1);
	buffer[0] = RHEA_EVENT_CPU_STATUS;
	this.sendGPUCommand ("E", buffer, 0, 0);
}

Rhea.prototype.sendStartDownloadDataAudit = function()
{
	var buffer = new Uint8Array(1);
	buffer[0] = RHEA_EVENT_READ_DATA_AUDIT; //108 l
	this.sendGPUCommand ("E", buffer, 0, 0);
}

Rhea.prototype.sendStartPosizionamentoMacina = function(macina_1o2, targetValue)
{
	//var	RHEA_EVENT_SET_POSIZIONE_MACINA = 122;				//'z'
	var buffer = new Uint8Array(4);
	buffer[0] = 122;
	buffer[1] = parseInt(macina_1o2);
	buffer[2] = ((targetValue & 0xFF00) >> 8);
	buffer[3] = (targetValue & 0x00FF);
	this.sendGPUCommand ("E", buffer, 0, 0);
}

Rhea.prototype.sendPartialDA3AndReturnAPromise = function(uno_di, num_tot, blockOffset, uint8array, startAt)
{
	var buffer = new Uint8Array(64+7);
	buffer[0] = RHEA_EVENT_SEND_PARTIAL_DA3;
	for (var i=0;i<64;i++)
		buffer[i+1] = uint8array[startAt+i];
	
	//buffer[65] = parseInt(uno_di);
	//buffer[66] = parseInt(num_tot);
	//buffer[67] = parseInt(blockOffset);
	rheaAddU16ToUint8Buffer (buffer, 65, uno_di);
	rheaAddU16ToUint8Buffer (buffer, 67, num_tot);
	rheaAddU16ToUint8Buffer (buffer, 69, blockOffset);
	this.sendGPUCommand ("E", buffer, 0, 0);
	
	this.partialDA3AckRcvd = -1;
	var me = this;
	return new Promise( function(resolve, reject) 
					{
						var timeoutMs = 5000;
						var check = function()	{
												if (me.partialDA3AckRcvd == blockOffset)
												{
													resolve ("OK");
												}
												else if ((timeoutMs -= 250) < 0)
												{
													resolve ("KO");
												}
												else
													setTimeout(check, 250);
											}

						setTimeout(check, 250);
					});				
}



/*********************************************************
 * webSocket_requestIDCodeAfterConnection
 *
 *	inviata automaticamente (se necessario) durante la connessione.
 *	Serve a chiedere alla SMU un idCode univoco da utilizzare da ora
 *	in poi durante ogni successiva riconnessione
 */
Rhea.prototype.webSocket_requestIDCodeAfterConnection = function()
{
	rheaLog("Rhea::webSocket requesting idCode"); 

	var buffer = new Uint8Array(4);
	buffer[0] = RHEA_CLIENT_INFO__API_VERSION;	//API version
	buffer[1] = RHEA_CLIENT_INFO__APP_TYPE;
	buffer[2] = RHEA_CLIENT_INFO__UNUSED2;
	buffer[3] = RHEA_CLIENT_INFO__UNUSED3;
	this.sendGPUCommand("I", buffer, 0, 0);
}

/*********************************************************
 * webSocket_identifyAfterConnection
 *
 *	inviata automaticamente durante la connessione.
 *	Serve ad identificarsi con la SMU
 */
Rhea.prototype.webSocket_identifyAfterConnection = function()
{
	rheaLog("Rhea::webSocket sending idCode[" +this.idCode_0 +"][" +this.idCode_1 +"][" +this.idCode_2 +"][" +this.idCode_3 +"]"); 

	var buffer = new Uint8Array(8);
	buffer[0] = RHEA_CLIENT_INFO__API_VERSION;	//API version
	buffer[1] = RHEA_CLIENT_INFO__APP_TYPE;
	buffer[2] = RHEA_CLIENT_INFO__UNUSED2;
	buffer[3] = RHEA_CLIENT_INFO__UNUSED3;
	
	buffer[4] = this.idCode_0;	//identification code MSB
	buffer[5] = this.idCode_1;
	buffer[6] = this.idCode_2;
	buffer[7] = this.idCode_3;
	this.sendGPUCommand("W", buffer, 0, 0);
}

/*********************************************************
 * filetransfer_startUpload
 *
 *	inizia l'upload di un file verso socketBridge
 *	
 */
Rhea.prototype.filetransfer_startUpload = function(rawData, usage, fileName, userValue, callback_onStart, callback_onProgress, callback_onEnd)
{
	//vediamo se ho uno slot libero
	for (var i=0; i<this.nFileTransfer; i++)
	{
		if (null == this.fileTransfer[i])
		{
			this.fileTransfer[i] = new RheaFileUpload(rawData, usage, fileName, userValue, callback_onStart, callback_onProgress, callback_onEnd);
			return;
		}
	}
	this.fileTransfer[this.nFileTransfer++] = new RheaFileUpload(rawData, usage, fileName, userValue, callback_onStart, callback_onProgress, callback_onEnd);
}

/*********************************************************
 * filetransfer_startDownload
 */
Rhea.prototype.filetransfer_startDownload = function(usage, userValue, callback_onStart, callback_onProgress, callback_onEnd)
{
	//vediamo se ho uno slot libero
	for (var i=0; i<this.nFileTransfer; i++)
	{
		if (null == this.fileTransfer[i])
		{
			this.fileTransfer[i] = new RheaFileDownload(usage, userValue, callback_onStart, callback_onProgress, callback_onEnd);
			return;
		}
	}
	this.fileTransfer[this.nFileTransfer++] = new RheaFileDownload(usage, userValue, callback_onStart, callback_onProgress, callback_onEnd);
}

Rhea.prototype.activateCPUBuzzer = function (numRepeat, beepLen_dSec, pauseBetweenBeeps_dSec)
{
	var buffer = new Uint8Array(4);
	buffer[0] = RHEA_EVENT_CPU_ACTIVATE_BUZZER;
	buffer[1] = parseInt(numRepeat);
	buffer[2] = parseInt(beepLen_dSec);
	buffer[3] = parseInt(pauseBetweenBeeps_dSec);
	this.sendGPUCommand ("E", buffer, 0, 0);
}

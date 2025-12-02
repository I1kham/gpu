var cpuLangName1 = "";
var cpuLangName2 = "";
var bGotoHMIAfterLavaggio = 0;
var bGotoHMIAfterBackOrSave = 0;

function startupOperation()
{
	syncDA3IfNeeded();
}

function syncDA3IfNeeded ()
{
	pleaseWait_freeText_appendText("da3 sync...");
	rhea.sendSyncDA3();
	setTimeout (function(){ syncDA3IfNeeded_onProgress();}, 600);
}

function syncDA3IfNeeded_onProgress ()
{
	if (currentCPUStatusID != -1 && currentCPUStatusID != 105)
	{
		pleaseWait_freeText_appendText("<br>");
		priv_askCPULangNames();
	}
	else
	{
		setTimeout (function(){ syncDA3IfNeeded_onProgress();}, 800);
		pleaseWait_freeText_appendText(".");
	}
}

function priv_askCPULangNames()
{
	pleaseWait_freeText_appendText("getting CPU lang names...<br>");
	rhea.ajax ("getCPULangNames", "").then( function(result)
	{
		var e = result.split("#");
		cpuLangName1 = e[0];
		cpuLangName2 = e[1];
		priv_askMachineTypeAndModel();
	})
	.catch( function(result)
	{
		console.log ("err[" +result +"]");
		cpuLangName1 = "lang 1";
		cpuLangName1 = "lang 2";
		setTimeout(function() { priv_askCPULangNames(); }, 200);
	});	
}

function priv_askMachineTypeAndModel()
{
	pleaseWait_freeText_appendText("getting machine type and model...<br>");
	rhea.ajax ("getMachineTypeAndModel", "").then( function(result)
	{
		var obj = JSON.parse(result);
		
console.log ("priv_askMachineTypeAndModelpriv_askMachineTypeAndModel"); console.log (obj);		
		
		//carico il da3
		pleaseWait_freeText_appendText("loading da3...<br>");
		da3 = new DA3(obj.mType, obj.mModel, obj.isInduzione, obj.gruppo, obj.aliChina, obj.motherboard, obj.customerID, obj.cap0, obj.cap1, obj.cap2, obj.cap3);
		if (obj.cpuArch !== undefined)
		{
			if (obj.cpuArch === "STM32")
				da3.cpuArch = "STM32";
		}
		da3.load();
	})
	.catch( function(result)
	{
		console.log ("ERR priv_askMachineTypeAndModel::[" +result +"]");
		setTimeout(function() { priv_askMachineTypeAndModel(); }, 200);
	});	
}

//***************** chiamata al termine del caricamento del da3
function onDA3Loaded(bResult)
{
	if (bResult == 0)
	{
		if(isTEMPESTIVECLOUD())
		{
			setTimeout ( function() { TempestiveCloud.relaod(); }, 1000);
		}
		else		
		{
			setTimeout ( function() { window.location = "index.html"; }, 1000);
		}
		return;
	}
	console.log ("onDA3Loaded:: num macine[" +da3.getNumMacine() +"]");
	//console.log ("onDA3Loaded:: isGrpVariflex?" + da3.isGruppoVariflex());
	//console.log ("onDA3Loaded:: isGrpMicro?" + da3.isGruppoMicro());
	
	//alcuni controlli sul da3
	//in caso di macchine non induzione, l'unica opzione per il cappuccinatore è il venturi
	if (!da3.isInduzione())
	{
		if (da3.read8(69) == 2)
			da3.write8(69,0);
	}
	
	
	//inizio il load delle GUI info
	guiInfo = new GUIInfo();
	
	if (isTEMPESTIVECLOUD()) 
	{
		// Skip GUI download when connecting from the cloud
		onGUIInfoLoaded();
	} 
	else 
		guiInfo.load();
}

//***************** chiamata al termine delle gui info
function onGUIInfoLoaded()
{
	pageProductQty_build();
	pageSelections_build();
	pageSingleSelection_build();
	pageTemperature_build();
	pleaseWait_hide();
	
	//chiedo se il manuale e' installato, nel qual caso abilito il relativo bottone
	if (!isTEMPESTIVECLOUD())
	{
		rhea.ajax ("isManInstalled", "").then( function(result)
		{
			if (result!="KO")
			{
				rheaShowElem(rheaGetElemByID("divBtnManual"));
				manualFolderName = result;
			}
		})
		.catch( function(result)
		{
			rheaShowElem(rheaGetElemByID("divBtnManual"));
		});
		
		if (da3.hasCapab_DEBUG_BTN())
		{
			var html = "<div class='UIButton bigButton3x4' data-caption='<b>D E B U G</b>' data-onclick='pageDebug_show()'></div>";
			rheaSetElemHTML (rheaGetElemByID("tab_mainmenu_tdManual"), html);
		}
	}
	
	
	//abilito menu "espresso calibration" se macchine==espresso
	//abilito menu "milker" se macchine==espresso
	if (da3.isEspresso())
	{
		rheaRemoveClassToElem(rheaGetElemByID("pageMainMenu_btnEspressoCalib"), "UIdisabled");
		if (da3.isGruppoVariflex())
			rheaRemoveClassToElem(rheaGetElemByID("pageMainMenu_btnMilker"), "UIdisabled");
	}
	
	//disabilito varie voci "instant" se la macchina non ha funzionalità solubili
	if (!da3.hasAnyInstant())
	{
		rheaAddClassToElem(rheaGetElemByID("pageMainMenu_btnInstantCalib"), "UIdisabled");
		rheaAddClassToElem(rheaGetElemByID("pageMainMenu_btnProdQty"), "UIdisabled");
	}
	
	if (da3.hasCapab_NO_PAYMENT_SYSTEM())
	{
		rheaHideElem (rheaGetElemByID("pageMainMenu_btnPayment"));
		if (da3.isCustomer_LUXOR())
			rheaHideElem (rheaGetElemByID("pageMainMenu_btnDataAudit"));
	}
	
	/*if (da3.isMachine_BaristaSuprema())
	{
		rheaHideElem (rheaGetElemByID("pageMainMenu_btnProdQty"));
		rheaHideElem (rheaGetElemByID("pagePleaseWait_milkerInduxWashing_1"));
		document.getElementById("pagePleaseWait_milkerInduxWashing_2").src = "img/milkerInduxClean02_b.jpg";
		document.getElementById("pagePleaseWait_milkerInduxWashing_3").src = "img/milkerInduxClean03_b.jpg";
		document.getElementById("pagePleaseWait_milkerInduxWashing_4").src = "img/milkerInduxClean04_b.jpg";
	}*/
	
	//da ora in poi, il currentTask.onTimer() viene chiamata una volta al secondo
	setInterval (function() { timeNowMSec+=1000; currentTask.onTimer(timeNowMSec); }, 1000)
	
	//se sono in modalità "SECO_CLOUD" disabilito alcune voci
	switch (rheaGetURLParamOrDefault("mode", "0"))
	{
		default:
			menuMode=0;
			break;
			
		case "1":		
		case "secocloud":
		case "2":
		case "tempestivecloud":	
			menuMode = 1;
			break;
	}
	console.log ("Menumode=" +menuMode);
	
	var bShowBtn_OLD_MENU_PROG = 0;
	switch (menuMode)
	{
		case 0:
			if (!da3.hasCapab_NO_PAYMENT_SYSTEM())
				bShowBtn_OLD_MENU_PROG = 1;
			break;
			
		case 1:
			rheaShowElem_TABLE(rheaGetElemByID("divBtnQuitAndSave"));
			break;
	}
	
	//su STM32, il btn OLD_MENU non lo facciamo + vedere
	if (da3.isCPUArch_STM32())
		bShowBtn_OLD_MENU_PROG = 0;
	
	if (bShowBtn_OLD_MENU_PROG)
		rheaShowElem_TABLE(rheaGetElemByID("divBtnOldMenuProg"));
	
	
	//di default mostro la pagina pageMainMenu. Se però in GET c'è specificata un'altra pagina..
	switch (rheaGetURLParamOrDefault("page", "pageMainMenu"))
	{
		case "pageMainMenu": 		pageMainMenu_show(); break;
		case "pageCleaning": 		pageCleaning_show(); break;
		case "pageMaintenance": 	pageMaintenance_show(); break;
		case "pageMiscellaneous": 	pageMiscellaneous_show(); break;
		case "pageProductQty": 		pageProductQty_show(); break;
		case "pageMilker": 			pageMilker_show(); break;
		case "pageFreevend": 		bGotoHMIAfterLavaggio=1; pageMiscellaneous_show(); pageMiscellaneous_showPageFV(); break;
		case "pageHappy": 			bGotoHMIAfterLavaggio=1; pageClock_show(); pageClock_showPageHappy(); break;
		case "pageSingleSel":		
			{
				var iSel = parseInt(rheaGetURLParamOrDefault("selNum","0"))
				if(iSel>0)
				{
					//var url = window.location.href.split("?")[0];
					//window.location.href=window.location.href.split("?")[0];
					//var idx = url.indexOf("?");
					//alert(url);
					pageSingleSelection_show(iSel);
				}
				
			}        
			break;
		case "pageCleaningFromQuick": 
			bGotoHMIAfterBackOrSave = 1;
			pageCleaning_show(); 
			break;

		case "pageCleaningSanitario": 
			//start del lavaggio sanitario e, alla fine, goto back to HMI
			pageCleaning_show(); 
			pageCleaning_startLavSanitario(8,0); 
			bGotoHMIAfterLavaggio = 1;
			break;
			
		case "pageMilkerDescaling":				//#4419
			//start del descaling milker e, alla fine, goto back to HMI
			pageCleaning_show(); 
			pageCleaning_startLavSanitario(164,0); 
			bGotoHMIAfterLavaggio = 1;
			break;
			
		case "pageSyrupCleaning":				//#4660
			//start del lavaggio sanitario del modulo sciroppi e, alla fine, goto back to HMI
			pageCleaning_show(); 
			pageCleaning_startLavSanitario(165,0); 
			bGotoHMIAfterLavaggio = 1;
			break;

		case "pageCleaningMilker": 
			//start del lavaggio milker e, alla fine, goto back to HMI
			pageCleaning_show(); 
			
			//prima di poter partire col lavaggio sanitario del milker, bisogna che la CPU non sia ancora in stato INI_CHECK
			bGotoHMIAfterLavaggio = 1;
			pleaseWait_show(); 
			pleaseWait_freeText_show();
			pleaseWait_freeText_setText("MILK MODULE CLEANING<br>Waiting for machine to be ready");
			setTimeout (waitCPUNoMoreInINI_CHECK_status_thenStartCleanMilker(3), 100);
			break;
			
		case "pageDescaling": 
			//start del descaling e, alla fine, goto back to HMI
			pageCleaning_show(); 
			pageCleaning_startLavSanitario(161,0); 
			bGotoHMIAfterLavaggio = 1;
			break;

		case "pageDataAudit": 
			//show partial DA e, alla fine, goto back to HMI
			pageDataAudit_show(0);
			break;
			
		case "pageUninstall": 
			//caso di "seconda disinstallazione"
			pageMaintenance_disinstall();
			currentTask.cpuStatus = 13;
			currentTask.isSecondUninstall = 1;
			currentTask.fase = 30;
			rhea.requestGPUEvent(RHEA_EVENT_CPU_STATUS);
			break;
			
		case "pageQuickProductQty":
			bGotoHMIAfterLavaggio = 1;
			pageProductQty_show();
			break;
	}	
}
	
//**** attende che lo stato di CPU diventi == READY e poi parte con il lavaggio sanitario
function waitCPUNoMoreInINI_CHECK_status_thenStartCleanMilker(howManyTimesLeft)
{
	pleaseWait_freeText_appendText(".");
	
	if (currentCPUStatusID == 23 || currentCPUStatusID == 24) //23==MILK_WASHING_VENTURI, 24=MILK_WASHING_INDUX
	{
		//la CPU è già in lavaggio milker, non devo aspettare alcun cambio di stato
		pageCleaning_startLavSanitario(5,0); 
	}
	else if (currentCPUStatusID == 2) //2==READY
	{	
		//la CPU è diventata ready, ma aspetto 2,3 secondi prima di comandare il lavaggio
		howManyTimesLeft--;
		if (howManyTimesLeft == 0)
		{
			pleaseWait_freeText_appendText ("<br>Cleaning is starting");
			pleaseWait_hide();
			//pageCleaning_startLavSanitario(5,1); 
			pageCleaning_action(5);
		}
		else
			setTimeout (function() { waitCPUNoMoreInINI_CHECK_status_thenStartCleanMilker(howManyTimesLeft); }, 1000);
		return;
	}
	else
	{
		//la CPU non è READY e nemmeno MILK_WASHING, aspetto che entri in uno dei 2 stati
		setTimeout (function() { waitCPUNoMoreInINI_CHECK_status_thenStartCleanMilker(3); }, 1000);
	}
}







/********************************************************
 * Task
 *
 * Questo è il template delle classi "task".
 * Tutte le classi "derivate", devono implementare i metodi "on"
 */
function TaskVoid()																{}
TaskVoid.prototype.onTimer 				= function(timeNowMsec)					{}
TaskVoid.prototype.onEvent_cpuStatus 	= function(statusID, statusStr, flag16)	{}
TaskVoid.prototype.onEvent_cpuMessage 	= function(msg, importanceLevel)		{ rheaSetDivHTMLByName("footer_C", msg); }
TaskVoid.prototype.onFreeBtn1Clicked	= function(ev)							{}
TaskVoid.prototype.onFreeBtn2Clicked	= function(ev)							{}
TaskVoid.prototype.onFreeBtnTrickClicked= function(ev)							{}


/**********************************************************
 * TaskTemperature
 */
function TaskTemperature()																
{ 
	this.nextTimeCheckMSec=0;
	this.isPopupX9Visible=false;
}
TaskTemperature.prototype.onEvent_cpuStatus 	= function(statusID, statusStr, flag16)	{}
TaskTemperature.prototype.onEvent_cpuMessage 	= function(msg, importanceLevel)		{ rheaSetDivHTMLByName("footer_C", msg); }
TaskTemperature.prototype.onFreeBtn1Clicked	= function(ev)
{
	if (this.isPopupX9Visible)
	{
		this.isPopupX9Visible=false;
		pleaseWait_hide();
		setTimeout(pageTemperature_askX9PIN(), 10);
	}
}
TaskTemperature.prototype.onFreeBtn2Clicked	= function(ev)
{
	if (this.isPopupX9Visible)
	{
		this.isPopupX9Visible=false;
		pleaseWait_hide();
		setTimeout(pageTemperature_onX9Changed_abort, 10);
	}	
}
TaskTemperature.prototype.onFreeBtnTrickClicked= function(ev)							{}
TaskTemperature.prototype.onTimer 				= function(timeNowMsec)
{
	if (timeNowMsec >= this.nextTimeCheckMSec)
	{
		this.nextTimeCheckMSec = timeNowMsec + 3000;
		rhea.ajax ("getVandT", "").then( function(result)
		{
			var data = JSON.parse(result);
			rheaSetDivHTMLByName ("pageTemperature_live", pageMaintenance_formatHTMLForTemperature(data));
		})
		.catch( function(result)
		{
		});		
	}
}
TaskTemperature.prototype.showPopupConfirmX9 = function()
{
	this.isPopupX9Visible=true;
	pleaseWait_show();
	pleaseWait_rotella_hide();
	pleaseWait_freeText_setText("Atenție, setarea ciclului prima cafea funcționează numai cu kitul piston inferior (cu fante laterale) și recipient cafea (cu gaură în partea inferioară) din care se poate scurge apa de spălare în tava de picurare în mod adecvat; confirmați că aceste componente au fost instalate??");
	pleaseWait_freeText_show();
	pleaseWait_btn1_setText("CONFIRMĂ");
	pleaseWait_btn1_show();
	pleaseWait_btn2_setText("NU");
	pleaseWait_btn2_show();
}


/**********************************************************
 * TaskMaintenance
 */
function TaskMaintenance()																{ this.nextTimeCheckMSec=0;}
TaskMaintenance.prototype.onEvent_cpuStatus 	= function(statusID, statusStr, flag16)	{}
TaskMaintenance.prototype.onEvent_cpuMessage 	= function(msg, importanceLevel)		{ rheaSetDivHTMLByName("footer_C", msg); }
TaskMaintenance.prototype.onFreeBtn1Clicked	= function(ev)								{}
TaskMaintenance.prototype.onFreeBtn2Clicked	= function(ev)								{}
TaskMaintenance.prototype.onFreeBtnTrickClicked= function(ev)							{}
TaskMaintenance.prototype.onTimer 				= function(timeNowMsec)
{
	if (timeNowMsec >= this.nextTimeCheckMSec)
	{
		this.nextTimeCheckMSec = timeNowMsec + 3000;
		rhea.ajax ("getVandT", "").then( function(result)
		{
			var data = JSON.parse(result);
			rheaSetDivHTMLByName ("pageMaintenance_volt", helper_intToFixedOnePointDecimal(data.v) +" V");
			rheaSetDivHTMLByName ("pageMaintenance_temperature", pageMaintenance_formatHTMLForTemperature(data));
		})
		.catch( function(result)
		{
		});		
	}
}

/**********************************************************
 * TaskCleaning
 */
function TaskCleaning (whichWashIN, isEspresso)
{
	this.timeStarted = 0;
	this.isEspresso = isEspresso;
	this.cpuStatus = 0;
	this.whichWash = whichWashIN;
	this.fase = 0;
	this.btn1 = 0;
	this.btn2 = 0;
	this.btn3 = 0;
	this.btnTrick = 0;
	this.prevFase = 99999;
	this.prevFaseBtn1 = 999;
	this.prevFaseBtn2 = 999;
	this.lavaggioConLiquidoDetergente = 0;
	this.nextTimeSanWashStatusCheckMSec = 0;
	
	rhea.sendGetCPUStatus();
}
TaskCleaning.prototype.onTimer = function (timeNowMsec)
{
	if (this.timeStarted == 0)
		this.timeStarted = timeNowMsec;
	var timeElapsedMSec = timeNowMsec - this.timeStarted;
	
	if (this.whichWash == 8)
	{
		if (da3.isGruppoMicro())
			this.priv_handleSanWashingMicro(timeElapsedMSec);
		else
			this.priv_handleSanWashingVFlex(timeElapsedMSec);
	}
	else if (this.whichWash == 5 || this.whichWash == 163)
	{
		if (da3.isCappuccinatoreInduzione())
			this.priv_handleMilkWashingIndux(timeElapsedMSec);
		else
			this.priv_handleMilkWashingVenturi(timeElapsedMSec);
	}
	else if (this.whichWash == 161)
	{
		this.priv_handleDescalingVFlex(timeElapsedMSec);
	}
	else if (this.whichWash == 164)
	{
		this.priv_handleDescalingMilker(timeElapsedMSec);			//#4419
	}
	else if (this.whichWash == 165)									//#4660
	{
		this.priv_handleSyrupCleaning(timeElapsedMSec);				//#4660
	}
	else
	{
		if (timeElapsedMSec < 2000)
			return;
		if (this.cpuStatus != 7) //7==manual washing
			pageCleaning_onFinished();
	}
}

TaskCleaning.prototype.onEvent_cpuStatus  = function(statusID, statusStr, flag16)	
{ 
	this.cpuStatus = statusID; 
	pleaseWait_header_setTextL (statusStr);
}

TaskCleaning.prototype.onEvent_cpuMessage = function(msg, importanceLevel)			
{ 
	if (this.whichWash == 8)
	{
		//siamo nel lavaggio sanitario. In questo caso, non voglio mostrare alcun msg di CPU
		//perchè c'è già la GUI a guidare l'utente. L'unico msg che voglio mostrare è il conto alla rovescia
		//che CPU fa durante lo scioglimento pastiglia
		var bProcessaMsg = 0;
		
		if (da3.isGruppoMicro())
		{
			if (this.fase == 4)
				bProcessaMsg = 1;
		}
		else
		{
			if (this.fase == 3 || this.fase == 5)
				bProcessaMsg = 1;
		}
		
		if (bProcessaMsg == 1)
		{
			//prendo solo i numeri del conto alla rovescia, il resto del msg non mi interessa
			var n = parseInt(msg.length);
			if (n >= 4)
			{
				msg = msg.slice (n-4);
				if (msg.slice(1,2) != ':')
					msg = "";
			}
		}
		else
			msg = "";
	}
	rheaSetDivHTMLByName("footer_C", msg); 
	pleaseWait_header_setTextR(msg); 
}

TaskCleaning.prototype.onFreeBtn1Clicked	= function(ev)							{ rhea.sendButtonPress(this.btn1); pleaseWait_btn1_hide(); pleaseWait_btn2_hide(); pleaseWait_btn3_hide(); pleaseWait_btnTrick_hide();}
TaskCleaning.prototype.onFreeBtn2Clicked	= function(ev)							{ rhea.sendButtonPress(this.btn2); pleaseWait_btn1_hide(); pleaseWait_btn2_hide(); pleaseWait_btn3_hide(); pleaseWait_btnTrick_hide();}
TaskCleaning.prototype.onFreeBtn3Clicked	= function(ev)							{ rhea.sendButtonPress(this.btn3); pleaseWait_btn1_hide(); pleaseWait_btn2_hide(); pleaseWait_btn3_hide(); pleaseWait_btnTrick_hide();}
TaskCleaning.prototype.onFreeBtnTrickClicked= function(ev)							{ rhea.sendButtonPress(this.btnTrick); pleaseWait_btn1_hide(); pleaseWait_btn2_hide(); pleaseWait_btnTrick_hide();}

TaskCleaning.prototype.priv_handleSanWashingMicro = function (timeElapsedMSec)
{
	//termino quando lo stato della CPU diventa != da SAN_WASHING
	if (timeElapsedMSec > 3000 && this.cpuStatus != 20) //20==sanitary washing
	{
		pageCleaning_onFinished();
		return;
	}

	//ogni tot mando una richiesta per conoscere lo stato attuale del lavaggio
	if (timeElapsedMSec < this.nextTimeSanWashStatusCheckMSec)
		return;
	this.nextTimeSanWashStatusCheckMSec += 2000;

	//periodicamente richiedo lo stato del lavaggio
	var me = this;
	rhea.ajax ("sanWashStatus", "")
		.then( function(result) 
		{
			var obj = JSON.parse(result);
			//console.log ("SAN WASH response: fase[" +obj.fase +"] b1[" +obj.btn1 +"] b2[" +obj.btn2 +"]");
			me.fase = parseInt(obj.fase);
			me.btn1 = parseInt(obj.btn1);
			me.btn2 = parseInt(obj.btn2);
			switch (me.fase)
			{
				case 0: pleaseWait_freeText_setText("Spălarea grupului nu este începută(sau terminată)"); break;	//Brewer Cleaning is not started (or ended)
				case 1: pleaseWait_freeText_setText("Spălarea grupului a început"); break; 	//Brewer Cleaning is started
				case 2: pleaseWait_freeText_setText("Grup în poziție"); break; 	//Brewer placed
				case 3: pleaseWait_freeText_setText("Introdu tableta și apasă START"); break; //Put pastille and push START
				case 4: pleaseWait_freeText_setText("Infuzie"); break; //Infusion
				case 5: pleaseWait_freeText_setText("Cicluri de spălare a grupului 1"); break; // Brewer cleaning cycles 1
				case 6: pleaseWait_freeText_setText("Cicluri de spălare a grupului 2"); break;	// Brewer cleaning cycles 2
				case 7: pleaseWait_freeText_setText("Cicluri de spălare a grupului 3"); break;	// Brewer cleaning cycles 3
				case 8: pleaseWait_freeText_setText("Cicluri de spălare a grupului 4"); break;	// Brewer cleaning cycles 4
				case 9: pleaseWait_freeText_setText("Cicluri de spălare a grupului 5"); break;	// Brewer cleaning cycles 56
				case 10: pleaseWait_freeText_setText("Cicluri de spălare a grupului 6"); break;	// HC_STEP_BRW_6
				
				case 11: pleaseWait_freeText_setText("Repetă spălarea?"); break;	//Repeat cleaning ?
				case 12: pleaseWait_freeText_setText("Grup așezat în poziția de curățat cu  peria, apasă CONTINUARE la final"); break;	//Brewer placed in brush position, press CONTINUE when finished
				case 13: pleaseWait_freeText_setText("Sari peste ultima cafea sau fă o cafea"); break; //Skip final coffee or make a coffee
				case 14: pleaseWait_freeText_setText("Furnizare cafea"); break; //Coffee delivery
				
				case 15:	//HC_STEP_MIXER_1
				case 16:	//HC_STEP_MIXER_2
				case 17:	//HC_STEP_MIXER_3
				case 18:	//HC_STEP_MIXER_4
					cleanMixNum = me.fase-14;
					if (me.isEspresso)
						cleanMixNum++;
					pleaseWait_freeText_setText("Clătire " +cleanMixNum); //rinsing
					break; 	
				default: pleaseWait_freeText_setText(""); break;
			}
			pleaseWait_freeText_show();
				
			if (me.fase != me.prevFase)
			{
				me.prevFase = me.fase;

				if (me.btn1 == 0)
					pleaseWait_btn1_hide();
				else
				{
					if (me.fase == 4)
					{
						//qui, la CPU invia il BTN 3 che serve per "debug" per saltare la fase di dissoluzione della tab.
						//Non mostro il tasto, ma uso il btn trick che è in semi trasparenza nell'angolo in alto a dx. Gli operatore
						//che conoscono il trucco, possono cliccare sul btn invisibile per skippare questa fase
						me.btnTrick = 3; //me.btn1;
						pleaseWait_btnTrick_show();	
					}
					else
					{
						var btnText = "BUTON " +me.btn1;
						switch (me.fase)
						{
							case 3:  btnText = "START"; break; //HC_STEP_TABLET
							case 11: btnText = "NU"; break; //HC_STEP_BRW_REPEAT
							case 12: btnText = "CONTINUĂ"; break; //HC_STEP_BRW_BRUSH_POSITION
							case 13: btnText = "SARI PESTE CAFEA"; break; //HC_STEP_BRW_SKIP_FINAL_COFFEE
						}
						pleaseWait_btn1_setText (btnText);
						pleaseWait_btn1_show();	
					}
				}
				
				if (me.btn2 == 0)
					pleaseWait_btn2_hide();
				else
				{
					var btnText = "BUTON " +me.btn2;
					switch (me.fase)
					{
						case 11: btnText = "DA"; break;//HC_STEP_BRW_REPEAT
						case 13: btnText = "FĂ O CAFEA"; break; //HC_STEP_BRW_SKIP_FINAL_COFFEE
					}
					pleaseWait_btn2_setText (btnText);
					pleaseWait_btn2_show();	
				}
			} //if (me.fase != me.prevFase)
		})
		.catch( function(result)
		{
			//console.log ("SANWASH: error[" +result +"]");
			pleaseWait_btn1_hide();
			pleaseWait_btn2_hide();
		});	
	
}

//------- Descaling Cappuccinatore Indux -------------
TaskCleaning.prototype.priv_handleDescalingMilker = function (timeElapsedMSec)		//#4419
{
	//termino quando lo stato della CPU diventa != da MILKER DESCALING
	if (timeElapsedMSec > 3000 && this.cpuStatus != 30) //30==milker descaling
	{
		pageCleaning_onFinished();
		return;
	}
	
	//ogni tot mando una richiesta per conoscere lo stato attuale del lavaggio
	if (timeElapsedMSec < this.nextTimeSanWashStatusCheckMSec)
		return;
	this.nextTimeSanWashStatusCheckMSec += 2000;	

	//periodicamente richiedo lo stato del lavaggio
	var me = this;
	rhea.ajax ("sanWashStatus", "")
		.then( function(result) 
		{
			var obj = JSON.parse(result);
			//console.log ("DESCALING MILKER response: fase[" +obj.fase +"] b1[" +obj.btn1 +"] b2[" +obj.btn2 +"]");
			me.fase = parseInt(obj.fase);
			me.btn1 = parseInt(obj.btn1);
			me.btn2 = parseInt(obj.btn2);
			var msg = "";
			switch (me.fase)
			{
				default: 	msg = "???"; break;
				case 1:		msg = ". Apasă CONTINUARE la final."; break;
				case 2:		msg = "Golire circuit hidraulic, te rog așteaptă…"; break;
				case 3:		msg = ". Apasă CONTINUARE la final."; break;
				case 4:		msg = "Conectează pompa submersibilă la recipientul cu soluție de detartrare. Apasă CONTINUARE la final."; break;
				case 5:		msg = "Umplere circuit hidraulic cu soluție de detartrare, te rog așteaptă…"; break;
				case 6:		msg = "Verificare nivel soluție de detartrare din rezervorul de aer. Apasă CONTINUARE la final."; break;
				case 7:		msg = "Începere umplere conducte hidraulice cu soluție de detartrare, te rog așteaptă…"; break;
				case 8:		msg = "Te rog așteaptă ca soluția de detartrare să acționeze..."; break;
				case 9:		msg = "Soluția de detartrare începe să curgă prin duze, te rog așteaptă…"; break;
				case 10:	msg = "start emptying and rinsing of the hydraulic circuit"; break;
				case 11:	msg = ". Apasă CONTINUARE la final."; break;
				case 12:	msg = "Golire circuit hidraulic, te rog așteaptă…"; break;
				case 13:	msg = ". Apasă CONTINUARE la final."; break;
				case 14:	msg = "Mută sursa de alimentare în recipientul de apă. Conectează pompa submersibilă la recipientul cu apă. Apasă CONTINUARE la final."; break;
				case 15:	msg = "Circuitul hidraulic se va umple cu apă, te rog așteaptă…"; break;
				case 16:	msg = "Verifică nivelul apei din rezervorul de aer. Apasă CONTINUARE la final."; break;
				case 17:	msg = "Apa se scurge prin duze."; break;
				case 18:	msg = "Furnizează apă și testează proba. Pune un pahar pentru a preleva proba de apă. Apasă CONTINUARE la final."; break;
				case 19:	msg = "Începe să faci să curgă apa pentru probă prin fiecare duză."; break;
				case 20:	msg = "Verifică proba prelevată. Apasă butonul CONTINUARE pentru a continua sau REPETĂ pentru a repeta pașii precedenți și a curăța corespunzător circuitul hidraulic."; break;
				case 21:	msg = "Procedura de detartrare s-a încheiat, apasă ÎNCHIDE pentru finalizare."; break;
			}
			
			//msg += "<br><br>DEBUG-FASE[" +me.fase +"] BTN1{" +me.btn1 +"} BTN2[" +me.btn2 +"]";
			
			pleaseWait_freeText_setText(msg);
			pleaseWait_freeText_show();
			
			if ( (me.fase != me.prevFase) || ( me.fase == me.prevFase && (me.btn1 != me.prevFaseBtn1 || me.btn2 != me.prevFaseBtn2) ))
			{
				me.prevFase = me.fase;
				me.prevFaseBtn1 = me.btn1;
				me.prevFaseBtn2 = me.btn2;

				
				if (me.btn1 == 0)
					pleaseWait_btn1_hide();
				else
				{
					var btnText = "BUTON " +me.btn1;
					switch (me.fase)
					{
						case 1: 
						case 3:
						case 4:
						case 6:
						case 10:
						case 11:
						case 13:
						case 14:
						case 16:
						case 18:
						case 20:
							btnText = "CONTINUĂ"; 
							break;
							
						case 21:
							btnText = "ÎNCHIDE"; 
							break;
					}
					pleaseWait_btn1_setText (btnText);
					pleaseWait_btn1_show();	
				}
				
				if (me.btn2 == 0)
					pleaseWait_btn2_hide();
				else
				{
					var btnText = "BUTON " +me.btn2;
					switch (me.fase)
					{
						case 10:
						case 20:
							btnText = "REPETĂ";
							break;
					}
					pleaseWait_btn2_setText (btnText);
					pleaseWait_btn2_show();	
				}				
			}				
		})
		.catch( function(result)
		{
			//console.log ("DESCALING MILKER: error[" +result +"]");
			//pleaseWait_btn1_hide();
			//pleaseWait_btn2_hide();
		});	
		
}

//------- Descaling Gruppo Caffe' -------------
TaskCleaning.prototype.priv_handleDescalingVFlex = function (timeElapsedMSec)
{
	//termino quando lo stato della CPU diventa != da DESCALING
	if (timeElapsedMSec > 3000 && this.cpuStatus != 26) //26==descaling
	{
		pageCleaning_onFinished();
		return;
	}
	
	//ogni tot mando una richiesta per conoscere lo stato attuale del lavaggio
	if (timeElapsedMSec < this.nextTimeSanWashStatusCheckMSec)
		return;
	this.nextTimeSanWashStatusCheckMSec += 2000;	

	//periodicamente richiedo lo stato del lavaggio
	var me = this;
	rhea.ajax ("sanWashStatus", "")
		.then( function(result) 
		{
			var obj = JSON.parse(result);
			//console.log ("DESCALING response: fase[" +obj.fase +"] b1[" +obj.btn1 +"] b2[" +obj.btn2 +"]");
			me.fase = parseInt(obj.fase);
			me.btn1 = parseInt(obj.btn1);
			me.btn2 = parseInt(obj.btn2);
			var msg = "";
			switch (me.fase)
			{
				default: 	msg = "???"; break;
				case 1:		msg = "Deschide robinetul boilerului. Apasă CONTINUARE la final."; break;
				case 2:		msg = "Golire circuit hidraulic, te rog așteaptă…"; break;
				case 3:		msg = "Închide robinetul boilerului. Apasă CONTINUARE la final."; break;
				case 4:		msg = "Conectează pompa submersibilă la recipientul cu soluție de detartrare. Apasă CONTINUARE la final."; break;
				case 5:		msg = "Umplere circuit hidraulic cu soluție de detartrare, te rog așteaptă…"; break;
				case 6:		msg = "Verificare nivel soluție de detartrare din rezervorul de aer. Apasă CONTINUARE la final."; break;
				case 7:		msg = "Începere umplere conducte hidraulice cu soluție de detartrare, te rog așteaptă…"; break;
				case 8:		msg = "Te rog așteaptă ca soluția de detartrare să acționeze..."; break;
				case 9:		msg = "Soluția de detartrare începe să curgă prin duze, te rog așteaptă…"; break;
				case 10:	msg = "Verifică lichidul ieșit din duze pentru a vedea dacă procesul de detartrare s-a încheiat cu succes."; break;
				case 11:	msg = "Deschide robinetul boilerului. Apasă CONTINUARE la final."; break;
				case 12:	msg = "Golire circuit hidraulic, te rog așteaptă…"; break;
				case 13:	msg = "Închide robinetul boilerului. Apasă CONTINUARE la final."; break;
				case 14:	msg = "Mută sursa de alimentare în recipientul de apă. Conectează pompa submersibilă la recipientul cu apă. Apasă CONTINUARE la final."; break;
				case 15:	msg = "Circuitul hidraulic se va umple cu apă, te rog așteaptă…"; break;
				case 16:	msg = "Verifică nivelul apei din rezervorul de aer. Apasă CONTINUARE la final."; break;
				case 17:	msg = "Apa se scurge prin duze."; break;
				case 18:	msg = "Furnizează apă și testează proba. Pune un pahar pentru a preleva proba de apă. Apasă CONTINUARE la final."; break;
				case 19:	msg = "Începe să faci să curgă apa pentru probă prin fiecare duză."; break;
				case 20:	msg = "Verifică proba prelevată. Apasă butonul CONTINUARE pentru a continua sau REPETĂ pentru a repeta pașii precedenți și a curăța corespunzător circuitul hidraulic."; break;
				case 21:	msg = "Procedura de detartrare s-a încheiat, apasă ÎNCHIDE pentru finalizare."; break;
			}
			
			//msg += "<br><br>DEBUG-FASE[" +me.fase +"] BTN1{" +me.btn1 +"} BTN2[" +me.btn2 +"]";
			
			pleaseWait_freeText_setText(msg);
			pleaseWait_freeText_show();
			
			if ( (me.fase != me.prevFase) || ( me.fase == me.prevFase && (me.btn1 != me.prevFaseBtn1 || me.btn2 != me.prevFaseBtn2) ))
			{
				me.prevFase = me.fase;
				me.prevFaseBtn1 = me.btn1;
				me.prevFaseBtn2 = me.btn2;

				
				if (me.btn1 == 0)
					pleaseWait_btn1_hide();
				else
				{
					var btnText = "BUTON " +me.btn1;
					switch (me.fase)
					{
						case 1: 
						case 3:
						case 4:
						case 6:
						case 10:
						case 11:
						case 13:
						case 14:
						case 16:
						case 18:
						case 20:
							btnText = "CONTINUĂ"; 
							break;
							
						case 21:
							btnText = "ÎNCHIDE"; 
							break;
					}
					pleaseWait_btn1_setText (btnText);
					pleaseWait_btn1_show();	
				}
				
				if (me.btn2 == 0)
					pleaseWait_btn2_hide();
				else
				{
					var btnText = "BUTON " +me.btn2;
					switch (me.fase)
					{
						case 10:
						case 20:
							btnText = "REPETĂ";
							break;
					}
					pleaseWait_btn2_setText (btnText);
					pleaseWait_btn2_show();	
				}				
			}				
		})
		.catch( function(result)
		{
			//console.log ("DESCALING: error[" +result +"]");
			//pleaseWait_btn1_hide();
			//pleaseWait_btn2_hide();
		});	
		
}

TaskCleaning.prototype.priv_handleSanWashingVFlex = function (timeElapsedMSec)
{
	//termino quando lo stato della CPU diventa != da SAN_WASHING
	if (timeElapsedMSec > 3000 && this.cpuStatus != 20) //20==sanitary washing
	{
		pageCleaning_onFinished();
		return;
	}

	//ogni tot mando una richiesta per conoscere lo stato attuale del lavaggio
	if (timeElapsedMSec < this.nextTimeSanWashStatusCheckMSec)
		return;
	this.nextTimeSanWashStatusCheckMSec += 2000;

	//periodicamente richiedo lo stato del lavaggio
	var me = this;
	rhea.ajax ("sanWashStatus", "")
		.then( function(result) 
		{
			var obj = JSON.parse(result);
			//console.log ("SAN WASH response: fase[" +obj.fase +"] b1[" +obj.btn1 +"] b2[" +obj.btn2 +"]");
			me.fase = parseInt(obj.fase);
			me.btn1 = parseInt(obj.btn1);
			me.btn2 = parseInt(obj.btn2);
			switch (me.fase)
			{
				default: msg = ""; break;
				case 0: msg = "Spălarea este activată"; break;	//Cleaning is active
				case 1: msg = "Pune tableta în grup și apasă CONTINUARE"; break; 	//Put the pastille in the brewer and press CONTINUE
				case 2: msg = "Grupul se închide"; break; 	//Brewer is closing
				case 3: msg = "Dizolvare tabletă 1/2, te rog așteaptă"; break; 	//Tablet dissolution 1/2, please wait
				case 4: msg = "Al doilea ciclu de dizolvare începe în curând"; break; 	//2nd dissolution cycle is about to starting
				case 5: msg = "Dizolvare tabletă 2/2, te rog așteaptă"; break; 	//Tablet dissolution 1/2, please wait

				case 6: msg = "1-a Spălare, te rog așteaptă 1/3"; break; 	//1st Cleaning, please wait 1/3
				case 7: msg = "1-a Spălare, activă 1/3"; break; 	//1st Cleaning, active 1/3
				case 8: msg = "1-a Spălare, te rog așteaptă 2/3"; break; 	//1st Cleaning, please wait 2/3
				case 9: msg = "1-a Spălare, activă 2/3"; break; 	//1st Cleaning, active 2/3
				case 10: msg = "1-a Spălare, te rog așteaptă 3/3"; break; 	//1st Cleaning, please wait 3/3
				case 11: msg = "1-a Spălare, activă 3/3"; break; 	//1st Cleaning, active 3/3

				case 12: msg = "Vrei să repeți ciclul de spălare 1/2 ?"; break; 	//Do you want to repeat clean cycle 1/2 ?
				
				case 13: msg = "Grupul se deplasează în poziția deschisă"; break; 	//Brewer is going into open position
				
				case 14: msg = "Curăță manual cu peria și la final apasă CONTINUARE"; break; 	//Please, perform a manual brush and then press CONTINUE when finished
				
				case 15: msg = "a 2-a Spălare, te rog așteaptă 1/6"; break; 	//2nd Cleaning, please wait 1/6
				case 16: msg = "a 2-a Spălare, activă 1/6"; break; 	//2nd Cleaning, active 1/6
				case 17: msg = "a 2-a Spălare, te rog așteaptă 2/6"; break; 	//2nd Cleaning, please wait 2/6
				case 18: msg = "a 2-a Spălare, activă 2/6"; break; 	//2nd Cleaning, active 2/6
				case 19: msg = "a 2-a Spălare, te rog așteaptă 3/6"; break; 	//2nd Cleaning, please wait 3/6
				case 20: msg = "a 2-a Spălare, activă 3/6"; break; 	//2nd Cleaning, active 3/6
				case 21: msg = "a 2-a Spălare, te rog așteaptă 4/6"; break; 	//2nd Cleaning, please wait 4/6
				case 22: msg = "a 2-a Spălare, activă 4/6"; break; 	//2nd Cleaning, active 4/6
				case 23: msg = "a 2-a Spălare, te rog așteaptă 5/6"; break; 	//2nd Cleaning, please wait 5/6
				case 24: msg = "a 2-a Spălare, activă 5/6"; break; 	//2nd Cleaning, active 5/6
				case 25: msg = "a 2-a Spălare, te rog așteaptă 6/6"; break; 	//2nd Cleaning, please wait 6/6
				case 26: msg = "a 2-a Spălare, activă 6/6"; break; 	//2nd Cleaning, active 6/6
				
				case 27: msg = "Vrei să repeți ciclul de spălare 2/2 ?"; break; 	//Do you want to repeat clean cycle 2/2 ?
				
				case 28: msg = "Vrei să sari peste ultima cafea?"; break; //Do you want to skip the final coffee ?
				case 29: msg = "Furnizare cafea, te rog așteaptă"; break; //Coffee delivery, please wait

				case 30: msg = "Clătire Grup"; break; //Brewer Rinsing 
				case 31: msg = "Clătire Mixer 1"; break; //Mixer 1 Rinsing
				case 32: msg = "Clătire Mixer 2"; break; //Mixer 2 Rinsing
				case 33: msg = "Clătire Mixer 3"; break; //Mixer 3 Rinsing 
				case 34: msg = "Spălare Grup Finalizată. Apasă ÎNCHIDE pentru a termina"; break; //Brewer Cleaning Done. Press CLOSE to exit
				case 35: msg = "Vrei să efectuezi o spălare MANUALĂ sau AUTOMATĂ?"; break; //Do you want to perform MANUAL or AUTOMATIC washing?
			}

			//msg += "<br><br>DEBUG SAN WASH response: fase[" +obj.fase +"] b1[" +obj.btn1 +"] b2[" +obj.btn2 +"]";
			pleaseWait_freeText_setText (msg);
			pleaseWait_freeText_show();
				
			if ( (me.fase != me.prevFase) || ( me.fase == me.prevFase && (me.btn1 != me.prevFaseBtn1 || me.btn2 != me.prevFaseBtn2) ))
			{
				me.prevFase = me.fase;
				me.prevFaseBtn1 = me.btn1;
				me.prevFaseBtn2 = me.btn2;

				if (me.btn1 == 0)
					pleaseWait_btn1_hide();
				else
				{
					if (me.fase == 3 || me.fase == 5)
					{
						//qui, la CPU dovrebbe inviare il BTN 3 che serve per "debug" per saltare la fase di dissoluzione della tab.
						//Non mostro il tasto, ma uso il btn trick che è in semi trasparenza nell'angolo in alto a dx. Gli operatore
						//che conoscono il trucco, possono cliccare sul btn invisibile per skippare questa fase
						me.btnTrick = 3;
						pleaseWait_btnTrick_show();	
					}
					else
					{
						var btnText = "BUTON " +me.btn1;
						switch (me.fase)
						{
							case 1:  btnText = "CONTINUĂ"; break;
							case 12: btnText = "NU"; break; //Do you want to repeat clean cycle 1/2 ?
							case 14:  btnText = "CONTINUĂ"; break;
							case 27: btnText = "NU"; break; //Do you want to repeat clean cycle 2/2 ?
							case 28: btnText = "SARI PESTE CAFEA"; break; //Do you want to skip the final coffee ?
							case 34: btnText = "ÎNCHIDE"; break; //Do you want to skip the final coffee ?
							case 35: btnText = "AUTOMAT"; break;
							
						}
						pleaseWait_btn1_setText (btnText);
						pleaseWait_btn1_show();	
					}
				}
				
				if (me.btn2 == 0)
					pleaseWait_btn2_hide();
				else
				{
					var btnText = "BUTON " +me.btn2;
					switch (me.fase)
					{
						case 12: btnText = "DA"; break; //Do you want to repeat clean cycle 1/2 ?
						case 27: btnText = "DA"; break; //Do you want to repeat clean cycle 2/2 ?
						case 28: btnText = "FĂ O CAFEA"; break; //Do you want to skip the final coffee ?
						case 35: btnText = "MANUAL"; break;
					}
					pleaseWait_btn2_setText (btnText);
					pleaseWait_btn2_show();	
				}
			} //if (me.fase != me.prevFase)
		})
		.catch( function(result)
		{
			//console.log ("SANWASH: error[" +result +"]");
			pleaseWait_btn1_hide();
			pleaseWait_btn2_hide();
		});	
	
}

TaskCleaning.prototype.priv_handleMilkWashingVenturi = function (timeElapsedMSec)
{
	//termino quando lo stato della CPU diventa != da LAVAGGIO_MILKER
	if (timeElapsedMSec > 3000 && this.cpuStatus != 23) //23==LAVAGGIO_MILKER
	{
		pageCleaning_onFinished();
		return;
	}

	//ogni tot mando una richiesta per conoscere lo stato attuale del lavaggio
	if (timeElapsedMSec < this.nextTimeSanWashStatusCheckMSec)
		return;
	this.nextTimeSanWashStatusCheckMSec += 2000;

	//periodicamente richiedo lo stato del lavaggio
	var me = this;
	rhea.ajax ("sanWashStatus", "")
		.then( function(result) 
		{
			var obj = JSON.parse(result);
			me.fase = parseInt(obj.fase);
			me.btn1 = parseInt(obj.btn1);
			me.btn2 = parseInt(obj.btn2);
			switch (me.fase)
			{
				case 1: pleaseWait_freeText_setText("Spălarea cappuccinatorului a început"); break;	//Milker Cleaning is started
				case 2: pleaseWait_freeText_setText("Încălzire pentru spălare"); break;	//Warming for cleaner
				case 3: pleaseWait_freeText_setText("Asteapta confirmare"); break;	//Wait for confirm
				case 4: pleaseWait_freeText_setText("Cicluri de spălare în curs (12)"); break;	//Doing cleaner cycles (12)
				case 5: pleaseWait_freeText_setText("Încălzire apă"); break;	//Warming for water
				case 6: pleaseWait_freeText_setText("Așteaptă a doua confirmare"); break;	//Wait for second confirm
				case 7: pleaseWait_freeText_setText("Cicluri de spălare în curs (12)"); break;	//Doing cleaner cycles (12)
				default: pleaseWait_freeText_setText(""); break;
			}
			pleaseWait_freeText_show();
			
			if (me.btn1 == 0)
				pleaseWait_btn1_hide();
			else
			{
				switch (me.fase)
				{
					default: pleaseWait_btn1_setText ("BUTON " +me.btn2); break;
					case 2:  pleaseWait_btn1_setText ("START"); break;
					case 3:
					case 5:  pleaseWait_btn1_setText ("CONTINUĂ"); break;
					case 6:  pleaseWait_btn1_setText ("CONFIRMĂ"); break;
				}
				pleaseWait_btn1_show();	
			}
			
			if (me.btn2 == 0)
				pleaseWait_btn2_hide();
			else
			{
				pleaseWait_btn2_setText ("BUTON " +me.btn2);
				pleaseWait_btn2_show();	
			}			
		})
		.catch( function(result)
		{
			//console.log ("SANWASH: error[" +result +"]");
			pleaseWait_btn1_hide();
			pleaseWait_btn2_hide();
		});	
	
}

TaskCleaning.prototype.priv_handleMilkWashingIndux = function (timeElapsedMSec)
{
	//termino quando lo stato della CPU diventa != da eVMCState_LAVAGGIO_MILKER_INDUX
	if (timeElapsedMSec > 3000 && this.cpuStatus != 24) //24==eVMCState_LAVAGGIO_MILKER_INDUX
	{
		rheaHideElem (rheaGetElemByID("pagePleaseWait_milkerInduxWashing"));
		pageCleaning_onFinished();
		return;
	}

	//ogni tot mando una richiesta per conoscere lo stato attuale del lavaggio
	if (timeElapsedMSec < this.nextTimeSanWashStatusCheckMSec)
		return;
	this.nextTimeSanWashStatusCheckMSec += 2000;

	//periodicamente richiedo lo stato del lavaggio
	var me = this;
	rhea.ajax ("sanWashStatus", "")
		.then( function(result) 
		{
			var obj = JSON.parse(result);
			me.fase = parseInt(obj.fase);
			me.btn1 = parseInt(obj.btn1);
			me.btn2 = parseInt(obj.btn2);
			
			//console.log ("priv_handleMilkWashingIndux() => fase=" +me.fase +", b1=" +me.btn1 +", b2=" +me.btn2 +", det_liquido=" +me.lavaggioConLiquidoDetergente);
			if (me.fase == 1)
			{
				//la fase 1 del milker indux è particolare, bisogna mostrare una specifica schermata con immagini di riepilogo.
				//A seconda che il lavaggio sia con pastiglia o con detergente liquido, le istruzioni cambiano
				
				pleaseWait_rotella_hide();
				rheaShowElem (rheaGetElemByID("pagePleaseWait_milkerInduxWashing"));
				pleaseWait_freeText_hide();

				if (me.lavaggioConLiquidoDetergente==1)
				{
					rheaSetDivHTMLByName ("pagePleaseWait_milkerInduxWashing_info", "Before starting the cleaning, please check that:<ol><li>water supply is available</li><li>waste tank is empty</li><li>the milk tube is in the cleaning jug</li></ol>");
					rheaHideElem(rheaGetElemByID("pagePleaseWait_milkerInduxWashing_1"));
					document.getElementById("pagePleaseWait_milkerInduxWashing_2").src = "img/milkerInduxClean01_334.jpg";
					document.getElementById("pagePleaseWait_milkerInduxWashing_3").src = "img/milkerInduxClean02.jpg";
					document.getElementById("pagePleaseWait_milkerInduxWashing_4").src = "img/milkerInduxClean03.jpg";
					if (me.btn1 != 0)
						me.btn1 = 12;
				}
				else
				{
					rheaSetDivHTMLByName ("pagePleaseWait_milkerInduxWashing_info", "Înainte de a începe spălarea, vă rugăm să verificați dacă:<ol><li>aparatul este conectat la alimentarea cu apă</li><li>recipientul de reziduuri este gol</li><li>tubul pentru lapte este introdus în vasul de curățare</li><li>vasul de curățare conține detergentul</li></ol>");
					rheaShowElem_TD(rheaGetElemByID("pagePleaseWait_milkerInduxWashing_1"));
					document.getElementById("pagePleaseWait_milkerInduxWashing_2").src = "img/milkerInduxClean02.jpg";
					document.getElementById("pagePleaseWait_milkerInduxWashing_3").src = "img/milkerInduxClean03.jpg";
					document.getElementById("pagePleaseWait_milkerInduxWashing_4").src = "img/milkerInduxClean04.jpg";
					if (me.btn1 != 0)
						me.btn1 = 10;
				}
			}
			else
			{
				me.btn3 = 0;
				rheaHideElem (rheaGetElemByID("pagePleaseWait_milkerInduxWashing"));
				pleaseWait_rotella_show();
				
				switch (me.fase)
				{					
				case 2:	//CIC_STEP_FILL_WATER
					var cond_uS = parseInt(obj.buffer8[1]) + 256*parseInt(obj.buffer8[2]);
					pleaseWait_freeText_setText("Încărcare vas de apă, te rog așteaptă. <br> Conductivitatea apei: {0} uS".translateLang(cond_uS)); //Water refilling in progress, please wait.<br>Water conducibility: {0} uS
					break;

				case 3:	//CIC_WAIT_FOR_TABLET_DISSOLVING
					var timeSec = parseInt(obj.buffer8[0]);
					pleaseWait_freeText_setText("Spălare în curs:  -{0}".translateLang(timeSec)); //Rinsing -{0}
					break;

				case 4:	//CIC_cleaning_PHASE1
				case 5:	//CIC_cleaning_PHASE2
					var ciclo_num = parseInt(obj.buffer8[0]);
					var ciclo_di = parseInt(obj.buffer8[3]);
					var cond_uS = parseInt(obj.buffer8[1]) + 256*parseInt(obj.buffer8[2]);
					pleaseWait_freeText_setText("Spălare {0} din {1} în curs, te rog așteaptă.<br>Conductivitatea apei: {2} uS".translateLang(ciclo_num,ciclo_di,cond_uS)); //Rinsing {0} of {1}, please wait.<br>Water conducibility: {2} uS
					break;

				case 6: //CIC_RINSING_PHASE3
					var ciclo_num = parseInt(obj.buffer8[0]);
					var ciclo_di = parseInt(obj.buffer8[3]);
					var cond_uS = parseInt(obj.buffer8[1]) + 256*parseInt(obj.buffer8[2]);
					pleaseWait_freeText_setText("Clătire {0} din {1} în curs, te rog așteaptă.<br>Conductivitatea apei: {2} uS".translateLang(ciclo_num,ciclo_di,cond_uS)); //Cleaning {0} of {1}, please wait.<br>Water conducibility: {2} uS
					break;
				
				case 7: //CIC_WAIT_FOR_MILK_TUBE
					var cond_uS_init_rif = parseInt(obj.buffer8[4]) + 256*parseInt(obj.buffer8[5]);		//#3318
					var cond_uS_finale   = parseInt(obj.buffer8[6]) + 256*parseInt(obj.buffer8[7]);		//#3318
					pleaseWait_freeText_setText("Programul de spălare s-a terminat.<br><br>Nu uita:<ul><li>golește recipient reziduuri</li><li>pune la loc tub pentru lapte</li></ul><br>Apasă ÎNCHIDE pentru a închide această fereastră și verifică dacă există lapte sau<br> ÎNCHIDE FĂRĂ SEL.LAPTE dacă nu vrei să activezi selecțiile cu lapte".translateLang(cond_uS_init_rif,cond_uS_finale)); //The cleaning procedure is finished.<br><Initial conducibility {0} and current conducibility {1},><br>Please, remember to:<ul><li>empty Waste tank</li><li>put the milk pipe back into position</li></ol><br>Press CLOSE to close this window
					break;

				case 9: //CIC_WAIT_CONF_LIQUID_DETERGENT
					pleaseWait_freeText_setText("adăugați detergent la vasul de lapte, pune o carafă în stația pentru pahare și apasă ÎNCEPUT<br><br><center><img src='img/milkerInduxClean05.jpg'></center>"); //Please fill liquid detergent and push START
					break;
					
				case 97: //CIC_TOO_MUCH_DETERGENTE
					pleaseWait_freeText_setText("ATENȚIE: prea mult detergent");
					break;
				
				case 98: //CIC_DET_TOO_LOW
					pleaseWait_freeText_setText("ATENȚIE: detergent insuficient, se recomandă repetarea procesului de spălare");
					break;

				case 99: //CIC_DET_TOO_AGGRESSIVE
					pleaseWait_freeText_setText("ATENȚIE: detergent prea agresiv");
					break;

				default:
					//qui non ci dovrebbe mai arrivare, ma nel caso...
					pleaseWait_freeText_setText("INDUX WASH response: fase[" +me.fase +"] b1[" +me.btn1 +"] b2[" +me.btn2 +"], buffer["+obj.buffer8[0] +"," +obj.buffer8[1] +"," +obj.buffer8[2] +"," +obj.buffer8[3] +"," +obj.buffer8[4] +"," +obj.buffer8[5] +"," +obj.buffer8[6] +"," +obj.buffer8[7] +"]");
					break;
				}
				
				pleaseWait_freeText_show();
			}
			
			
			if (me.btn1 == 0)
				pleaseWait_btn1_hide();
			else
			{
				switch (me.fase)
				{
				default: 	pleaseWait_btn1_setText (me.btn1); break;
				case 1:
					if (me.lavaggioConLiquidoDetergente==1)
						pleaseWait_btn1_setText ("CONTINUĂ");
					else
						pleaseWait_btn1_setText ("START");
					break;
				case 7:		pleaseWait_btn1_setText ("ÎNCHIDE"); break;
				case 9:		pleaseWait_btn1_setText ("START"); break;
				}
				pleaseWait_btn1_show();	
			}
			
			if (me.btn2 == 0)
				pleaseWait_btn2_hide();
			else
			{
				switch (me.fase)
				{
				default: pleaseWait_btn2_setText (me.btn2); break;
				case 1:	 pleaseWait_btn2_setText ("ANULEAZĂ"); break;
				case 7:	 pleaseWait_btn2_setText ("ÎNCHIDE FĂRĂ SEL. LAPTE"); break;
				}
				pleaseWait_btn2_show();	
			}	

			if (me.btn3 == 0)
				pleaseWait_btn3_hide();
			else
			{
				switch (me.fase)
				{
				case 1:	 pleaseWait_btn3_setText ("CLEAN USING `LIQUID DETERGENT`"); break;
				}
				pleaseWait_btn3_show();	
			}			
		})
		.catch( function(result)
		{
			console.log ("INDUX WASH: error[" +result +"]");
			pleaseWait_btn1_hide();
			pleaseWait_btn2_hide();
			pleaseWait_btn3_hide();
			rheaHideElem (rheaGetElemByID("pagePleaseWait_milkerInduxWashing"));
		});		
	}



/**********************************************************
 * TaskCalibMotor
 */
function TaskCalibMotor()
{
	this.timeStarted = 0;
	this.what = 0;  //0==nulla, 1=calib motore prodott, 2=calib macina
	this.fase = 0;
	this.value = 0;
	this.cpuStatus = 0;
	this.bAlsoCalcImpulses = 1;
}

TaskCalibMotor.prototype.startCalibrazioneMacina = function (macina1o2, bAlsoCalcImpulses)
{
	console.log ("TaskCalibMotor.startCalibrazioneMacina => macina=" +macina1o2 +", calcImp=" +bAlsoCalcImpulses);
	var motor = 10 + macina1o2;
	this.startMotorCalib(motor);
	this.bAlsoCalcImpulses = bAlsoCalcImpulses;
}


TaskCalibMotor.prototype.startMotorCalib = function (motorIN) //motorIN==11 per macina1, 12 per macina2
{
	this.bAlsoCalcImpulses = 0;
	this.timeStarted = 0;
	this.fase = 0;
	this.motor = motorIN;
	this.impulsi = 0;
	this.value = 0;
	this.amIAskingForVGrindPos = 0;
	
	pleaseWait_calibration_varigrind_hide();
	pleaseWait_calibration_motor_hide();
	pleaseWait_calibEVPump_hide();
	
	this.what = 0;
	if (motorIN >= 11 && motorIN < 20)
		this.what = 2; //è una macina
	else
		this.what = 1;
}

TaskCalibMotor.prototype.startCalibrazioneEVPump = function (ev1_5, speed0_2)
{
	pleaseWait_calibration_varigrind_hide();
	pleaseWait_calibration_motor_hide();
	pleaseWait_calibEVPump_hide();
	
	
	this.timeStarted = 0;
	this.ev1_5 = ev1_5;
	this.speed0_2 = speed0_2;
	this.cc_sec = 0;
	
	this.fase = 0;
	this.what = 3;
}

TaskCalibMotor.prototype.showPopupCalibrazioneEVPump = function (ev1_5)
{
	this.timeStarted = 0;
	this.ev1_5 = ev1_5;
	this.fase = 0;
	this.what = 4;

	pleaseWait_show();
	pleaseWait_calibration_varigrind_hide();
	pleaseWait_calibration_motor_hide();
	pleaseWait_calibEVPump_hide();
	pleaseWait_rotella_hide();
	
	var v1 = helper_intToFixedOnePointDecimal( da3.getFattCalibPompeSolubile(ev1_5,0) );
	var v2 = helper_intToFixedOnePointDecimal( da3.getFattCalibPompeSolubile(ev1_5,1) );
	var v3 = helper_intToFixedOnePointDecimal( da3.getFattCalibPompeSolubile(ev1_5,2) );
	
	rheaSetDivHTMLByName ("pagePleaseWait_calibEVPump_low", v1);
	rheaSetDivHTMLByName ("pagePleaseWait_calibEVPump_norm", v2);
	rheaSetDivHTMLByName ("pagePleaseWait_calibEVPump_high", v3);
	
	pleaseWait_calibEVPump_show();
	pleaseWait_btn1_setText("ANULEAZĂ");
	pleaseWait_btn1_show();
}


TaskCalibMotor.prototype.onEvent_cpuStatus 	= function(statusID, statusStr, flag16)	{ this.cpuStatus = statusID;}
TaskCalibMotor.prototype.onEvent_cpuMessage = function(msg, importanceLevel)		{ rheaSetDivHTMLByName("footer_C", msg); pleaseWait_header_setTextR(msg);}
TaskCalibMotor.prototype.onTimer = function (timeNowMsec)
{
	if (this.timeStarted == 0)
		this.timeStarted = timeNowMsec;
	var timeElapsedMSec = timeNowMsec - this.timeStarted;

	switch (this.what)
	{
		case 1: this.priv_handleCalibProdotto(timeElapsedMSec); break;
		case 2: this.priv_handleCalibMacina(timeElapsedMSec); break;
		case 3: this.priv_handleCalibEvPump(timeElapsedMSec); break;
		case 4: this.priv_handlePopupCalibrazioneRVPump(timeElapsedMSec); break;
	}
}

TaskCalibMotor.prototype.onFreeBtn1Clicked	= function(ev)
{ 
	switch (this.what)
	{
	case 1: //priv_handleCalibProdotto
		if (this.fase == 30)	{ pleaseWait_btn1_hide(); this.fase = 40; }
		break;
		
	case 2: //priv_handleCalibMacina
		if (this.fase == 1)		{ pleaseWait_btn1_hide(); pleaseWait_btn2_hide(); this.fase = 2; }
		else if (this.fase==21) { pleaseWait_btn1_hide(); pleaseWait_btn2_hide(); this.fase=30; }
		else if (this.fase==41) { pleaseWait_btn1_hide(); pleaseWait_btn2_hide(); this.fase=50; }
		break;		

	case 3: //priv_handleCalibEvPump
		if (this.fase == 30)	{ pleaseWait_btn1_hide(); this.fase = 40; }
		break;		

	case 4: //priv_handleCalibEvPump
		if (this.fase == 0)	{ pleaseWait_btn1_hide(); this.fase = 200; }
		break;		
	}
}

TaskCalibMotor.prototype.onFreeBtn2Clicked	= function(ev)
{ 
	switch (this.what)
	{
	case 1: //priv_handleCalibProdotto
		break;
		
	case 2: //priv_handleCalibMacina
		if (this.fase == 1)		{ pleaseWait_btn1_hide(); pleaseWait_btn2_hide(); this.fase = 200; break; }
		break;		
	}
}
TaskCalibMotor.prototype.onFreeBtnTrickClicked= function(ev)							{}

TaskCalibMotor.prototype.priv_handleCalibProdotto = function (timeElapsedMSec)
{
	var TIME_ATTIVAZIONE_dSEC = 30;
	
	var me = this;
console.log ("TaskCalibMotor::fase[" +me.fase +"]");
	switch (this.fase)
	{
	case 0:
		me.fase = 10;
		pleaseWait_show();
		pleaseWait_calibration_show();
		pleaseWait_calibration_setText("Te rog așteaptă, motorul funcționează"); //Please wait while motor is running
		rhea.ajax ("runMotor", { "m":me.motor, "d":TIME_ATTIVAZIONE_dSEC, "n":2, "p":10}).then( function(result)
		{
			setTimeout ( function() { me.fase=20; }, TIME_ATTIVAZIONE_dSEC*2*100 - 1000);
		})
		.catch( function(result)
		{
			//console.log ("TaskCalibMotor:: error fase 0 [" +result +"]");
			me.fase = 200;
		});					
	break;
		
	case 10:
		break;
		
	case 20:
		me.fase = 30;
		
		//mostro la riga che chiede se si vuole rifare il movimento del motore
		pleaseWait_calibration_motor_show();
		pleaseWait_calibration_varigrind_hide();		

		
		pleaseWait_calibration_setText("La final, introdu cantitatea în grame obținută la ultima măcinare și apasă CONTINUARE"); //Please enter the quantity, then press CONTINUE
		pleaseWait_calibration_num_setValue(0);
		pleaseWait_calibration_num_show();
		pleaseWait_btn1_setText("CONTINUĂ");
		pleaseWait_btn1_show();
		break;
		
	case 30: //attende che l'utente prema uno dei pulsanti visibili a video
		break;
		
	case 40:
		pleaseWait_calibration_motor_hide();
		me.value = pleaseWait_calibration_num_getValue();
		if (parseFloat(me.value) == 0)
		{
			pleaseWait_calibration_setText("Valoare invalidă");
			me.fase = 20;
			break;
		}
	
		pleaseWait_calibration_setText("Salvare valoare ...");
		me.value = pleaseWait_calibration_num_getValue();
		me.gsec = parseInt( Math.round(me.value / (TIME_ATTIVAZIONE_dSEC*0.2)) );
		pleaseWait_calibration_num_hide();
		
		//console.log ("TaskCalibMotor::[40] motor[" +me.motor +"] value[" +me.value +"] g/sec[" +me.gsec +"]");
		
		rhea.ajax ("setFattoreCalib", { "m":me.motor, "v":me.gsec}).then( function(result)
		{
			me.fase = 199;
		})
		.catch( function(result)
		{
			//console.log ("TaskCalibMotor:: error fase 40 [" +result +"]");
			me.fase = 200;
		});		
		break;

		
	case 199:
		//console.log ("TaskCalibMotor::[199] motor[" +me.motor +"] value[" +me.value +"] g/sec[" +me.gsec +"]");
		da3.setCalibFactorGSec(me.motor, me.gsec);
		var v = helper_intToFixedOnePointDecimal( da3.getCalibFactorGSec(me.motor) );
		rheaSetDivHTMLByName("pageCalibration_m" +me.motor, v +"&nbsp;gr/sec");
		me.fase = 200;
		break;
		
	case 200:
		me.what = 0;
		pleaseWait_btn1_hide();
		pleaseWait_calibration_hide();
		pageCalibration_onFinish();
		break;
	}
}


TaskCalibMotor.prototype.priv_handlePopupCalibrazioneRVPump = function (timeElapsedMSec)
{
	switch (this.fase)
	{
	case 0:	//attendo che utente prema qualcosa
		break;
		
	case 200: //fine
		this.what = 0;
		pleaseWait_calibEVPump_hide();
		pleaseWait_hide();		
		break;
	}
}

TaskCalibMotor.prototype.priv_handleCalibEvPump = function (timeElapsedMSec)
{
	var TIME_ATTIVAZIONE_dSEC = 100;
	var NUM_ATTIVAZIONI = 2;
	var me = this;
//console.log ("TaskCalibMotor::fase[" +me.fase +"], ev=" +me.ev1_5 +", speed="+me.speed0_2);
	switch (this.fase)
	{
	case 0:
		me.fase = 9;
		me.runPumpRequestStarted_msec = timeElapsedMSec;
		pleaseWait_show();
		pleaseWait_calibration_show();
		pleaseWait_calibration_setText("Please wait while pump is running"); //Please wait while pump is running
		rhea.ajax ("runPump", { "ev":me.ev1_5, "speed":me.speed0_2, "d":TIME_ATTIVAZIONE_dSEC, "n":NUM_ATTIVAZIONI, "p":40}).then( function(result)
		{
			me.fase=10;
		})
		.catch( function(result)
		{
//console.log ("TaskCalibMotor:: error fase 0 [" +result +"]");
			me.fase = 200;
		});					
	break;
		
	case 9:
		break;
		
	case 10:	//attendo di vedere CPU passare in PREPARAZIONE_BEVANDA
		if (timeElapsedMSec - me.runPumpRequestStarted_msec > 3000)
			me.fase = 200;
		if (this.cpuStatus == 3)
			this.fase = 11;
		break;
		
	case 11:	//attendo di vedere CPU esca da PREPARAZIONE_BEVANDA
		if (this.cpuStatus != 3)
			this.fase = 20;
		break;
		
	case 20:
		me.fase = 30;
		pleaseWait_calibration_setText("Please enter the quantity in grams of the water dispensed, then press CONTINUE"); //Please enter the quantity in grams of the water dispensed, then press CONTINUE
		pleaseWait_calibration_num_setValue(0);
		pleaseWait_calibration_num_show();
		pleaseWait_btn1_setText("CONTINUĂ");
		pleaseWait_btn1_show();
		break;
		
	case 30: //attende che l'utente prema uno dei pulsanti visibili a video
		break;
		
	case 40:
		pleaseWait_calibration_motor_hide();
		me.value = pleaseWait_calibration_num_getValue();
		if (parseFloat(me.value) == 0)
		{
			pleaseWait_calibration_setText("Valoare invalidă");
			me.fase = 20;
			break;
		}
	
		pleaseWait_calibration_setText("Salvare valoare ...");
//console.log ("me.value=" +me.value +", time=" +TIME_ATTIVAZIONE_dSEC);
		me.cc_sec = parseInt( Math.floor( (me.value * 10) / (TIME_ATTIVAZIONE_dSEC*NUM_ATTIVAZIONI) ) );
		pleaseWait_calibration_num_hide();
		
//console.log ("TaskCalibMotor::[40] ev:" +me.ev1_5 +", speed:" +me.speed0_2 +", cc/sec:" +me.cc_sec);
		
		me.fase = 199;
		break;

		
	case 199:
		da3.setFattCalibPompeSolubile(me.ev1_5, me.speed0_2, me.cc_sec);
		me.fase = 200;
		break;
		
	case 200:
		me.what = 0;
		pleaseWait_btn1_hide();
		pleaseWait_calibration_hide();
		pageCalibration_onFinishWater();
		break;
	}
}


TaskCalibMotor.prototype.priv_handleCalibMacina = function (timeElapsedMSec)
{
	var TIME_ATTIVAZIONE_dSEC;
	
	if (da3.isMachine_rhTT1Plus())
	{
		TIME_ATTIVAZIONE_dSEC = 25;
		if (da3.read8(7554) == 0)
			TIME_ATTIVAZIONE_dSEC = 18;
	}
	else
	{
		TIME_ATTIVAZIONE_dSEC = 60;
		if (da3.read8(7554) == 0)
			TIME_ATTIVAZIONE_dSEC = 40;
	}		
	
	//console.log ("Tempo attivazione macina [" +TIME_ATTIVAZIONE_dSEC +"] dsec");
	
	var me = this;
	//console.log ("TaskCalibMotor::fase[" +me.fase +"]");
	switch (this.fase)
	{
	case 0:
		me.fase = 1;
		pleaseWait_show();
		pleaseWait_calibration_show();
		pleaseWait_calibration_setText("Scoate grup și apasă CONTINUARE. Fii pregătit să preiei cafeaua"); //Please remove the brewer, then press CONTINUE AND BE PREPARED TO CATCH COOFEE
		pleaseWait_btn1_setText("CONTINUĂ");
		pleaseWait_btn1_show();
		pleaseWait_btn2_setText("ANULEAZĂ");
		pleaseWait_btn2_show();	
		uiStandAloneVarigringTargetPos.setValue(0);
		break;
		
	case 1:	//attendo btn CONTINUE
		break;
		
	case 2: //verifico che il gruppo sia scollegato, altrimenti goto 0
		rhea.ajax ("getGroupState", "").then( function(result)
		{
			//console.log ("TaskCalibMotor, grpState[" +result +"]");
			if (result=="0")
				me.fase = 10;
			else
				me.fase = 0;
		})
		.catch( function(result)
		{
			me.fase = 0;
		});			
		me.fase = 3;
		break;
		
	case 3:	//attendo risposta CPU
		break;
		
	case 10:  //attivo le macinate
		me.fase = 11;
		pleaseWait_calibration_setText("Te rog așteaptă, motorul funcționează"); //Please wait while motor is running
		
//console.log ("TaskCalibMotor.priv_handleCalibMacina() => runMotor, m="  +me.motor);
		rhea.ajax ("runMotor", { "m":me.motor, "d":TIME_ATTIVAZIONE_dSEC, "n":2, "p":10}).then( function(result)
		{
			setTimeout ( function() { me.fase=20; }, TIME_ATTIVAZIONE_dSEC*2*100 - 1000);
		})
		.catch( function(result)
		{
			me.fase = 200;
		});					
	break;
		
	case 11: //attendo fine macinate
		break;
		
	case 20:
		me.fase = 21;
		
		//mostro le regolazioni per il gruppo caffè
		pleaseWait_calibration_motor_hide();
		pleaseWait_calibration_varigrind_show();		
		pleaseWait_calibEVPump_hide();
		
		pleaseWait_calibration_setText("La final, introdu cantitatea în grame obținută la ultima măcinare și apasă CONTINUARE"); //Please enter the quantity, then press CONTINUE
		pleaseWait_calibration_num_setValue(0);
		pleaseWait_calibration_num_show();
		pleaseWait_btn1_setText("CONTINUĂ");
		pleaseWait_btn1_show();
		break;
		
	case 21: //attendo pressione di continue per terminare, oppure GRIND AGAIN o SET per calibrare l'apertura del vgrind
		
		//periodicamente, chiedo la posizione attuale della macina del VGrind
		if (da3.isGruppoVariflex())
		{
			if (!me.amIAskingForVGrindPos)
			{
				me.amIAskingForVGrindPos = 1;
				rhea.ajax ("getPosMacina", {"m":(me.motor-10)}).then( function(result)
				{
					var obj = JSON.parse(result);
					rheaSetDivHTMLByName("pagePleaseWait_calibration_1_vg", obj.v);
					if (uiStandAloneVarigringTargetPos.getValue() == 0)					
						uiStandAloneVarigringTargetPos.setValue(obj.v)
					me.amIAskingForVGrindPos = 0;
				})
				.catch( function(result)
				{
					me.amIAskingForVGrindPos = 0;
				});		
			}
		}	
		break;
		
		
	case 25:	//qui ci andiamo se siamo in fase 21 e l'utente preme il btn SET per impostare una nuova apertura del VGrind
		pleaseWait_calibration_setText("Te rog așteaptă ca varigrind să își ajusteze poziția"); //Please wait while the varigrind is adjusting its position
		rhea.sendStartPosizionamentoMacina((me.motor-10), uiStandAloneVarigringTargetPos.getValue());
		me.fase = 26;
		break;
		
	case 26:	me.fase = 27; break;
	case 27:	me.fase = 28; break;
	case 28:
		if (da3.isGruppoVariflex())
		{
			rhea.ajax ("getPosMacina", {"m":(me.motor-10)}).then( function(result)
			{
				var obj = JSON.parse(result);
				rheaSetDivHTMLByName("pagePleaseWait_calibration_1_vg", obj.v);
				pleaseWait_calibration_setText("Te rog așteaptă ca varigrind să își ajusteze poziția" +"  [" +obj.v +"]"); //Please wait while the varigrind is adjusting its position  [current pos]
			})
			.catch( function(result)
			{
			});	
		}
		me.fase = 29;
		break;	

	case 29:
		//a questo punto CPU dovrebbe essere in stato 102 e dovrebbe rimanerci fino a fine operazione
		if (me.cpuStatus != 102 && me.cpuStatus != 101)
		{
			//ho finito, torno alla schermata dove è possibile macinare di nuovo, inputare i gr e calibrare il vgrind
			me.fase = 20;
		}
		else
			me.fase = 28;		
		break;
		
	case 30:
		me.value = pleaseWait_calibration_num_getValue();
		if (parseFloat(me.value) == 0)
		{
			pleaseWait_calibration_setText("Valoare invalidă");
			me.fase = 20;
			break;
		}
		
		pleaseWait_calibration_varigrind_hide();
		pleaseWait_calibration_setText("Salvare valoare ...");
		me.gsec = parseInt( Math.round(me.value / (TIME_ATTIVAZIONE_dSEC*0.2)) );
		pleaseWait_calibration_num_hide();
		
//console.log	("TaskCalibMotor.priv_handleCalibMacina: fase 30"); 
		da3.setCalibFactorGSec(me.motor, me.gsec);
		//var v = helper_intToFixedOnePointDecimal( da3.getCalibFactorGSec(me.motor) );
		//rheaSetDivHTMLByName("pageCalibration_m" +me.motor, v +"&nbsp;gr/sec");

//console.log	("TaskCalibMotor.priv_handleCalibMacina: setFattoreCalib m=" +me.motor +",v=" +me.gsec);		
		rhea.ajax ("setFattoreCalib", { "m":me.motor, "v":me.gsec}).then( function(result)
		{
			me.fase = 40;
		})
		.catch( function(result)
		{
			me.fase = 200;
		});		
		break;
		
	case 40: //chiedo di rimettere a posto il gruppo
		pleaseWait_calibration_setText("Repoziționează grup cafea și apasă CONTINUARE"); //Place the brewer into position, then press CONTINUE
		pleaseWait_btn1_show();
		me.fase = 41;
		break;
		
	case 41:
		break;
		
	case 50: //verifico che il gruppo sia collegato, altrimenti goto 40
		rhea.ajax ("getGroupState", "").then( function(result)
		{
			//console.log ("TaskCalibMotor, grpState[" +result +"]");
			if (result=="1")
				me.fase = 60;
			else
				me.fase = 40;
		})
		.catch( function(result)
		{
			me.fase = 40;
		});			
		me.fase = 51;
		break;
		
	case 51://attendo risposta CPU
		break;
		
	case 60: //gruppo è stato ricollegato, procedo con il calcolo impulsi se richiesto
		if (me.bAlsoCalcImpulses==0)
			me.fase = 200;
		else
		{
			me.fase = 65;
			pleaseWait_calibration_show();
			pleaseWait_calibration_setText("Calculul impulsului este în curs, așteaptă"); //Impulse calculation in progress, please wait
			rhea.ajax ("startImpulseCalc", { "m":me.motor, "v":me.value}).then( function(result)
			{
				//me.fase = 70;
				setTimeout ( function() { me.fase = 70; }, 3000);
			})
			.catch( function(result)
			{
				me.fase = 60;
			});
		}
		break;		
			
	case 65: //attendo risposta CPU
		break;
		
	case 70: //cpu sta facendo i conti degli impulsi, mando query per sapere come sta
		me.fase = 71;
		rhea.ajax ("queryImpulseCalcStatus", "").then( function(result)
		{
			//console.log ("queryImpulseCalc::result[" +result +"]");
			var obj = JSON.parse(result);
			if (parseInt(obj.v) > 0)
			{
				me.impulsi= parseInt(obj.v);
				me.fase = 190;
			}
			else
				me.fase = 70;
		})
		.catch( function(result)
		{
			me.fase = 70;
		});			
		break;
		
	case 71:
		break;
		
		
	case 190: //devo memorizzare gli impulsi ricevuti nel da3??
		da3.setImpulsi(me.motor, me.impulsi);

		var s = me.impulsi.toString();
		while (s.length < 3) s = "0" +s;		
		pleaseWait_calibration_setText("Impulse: " +s.substr(0,1) +"." +s.substr(1,2));
		
		me.fase = 191;
		break;
		
	case 191: me.fase++; break;
	case 192: me.fase++; break;
	case 193: me.fase++; break;
	case 194: me.fase++; break;
	case 195: me.fase=200; break;
		
	case 200:
		me.what = 0;
		pleaseWait_btn1_hide();
		pleaseWait_calibration_hide();
		pageCalibration_onFinish();
		break;
	}
}

/**********************************************************
 * TaskTestSelezione
 */
function TaskTestSelezione(selNum, iAttuatore)
{
	this.timeStarted = 0;
	this.selNum = selNum;
	this.iAttuatore = iAttuatore;
	this.cpuStatus = 0;
	this.fase = 0;
}

TaskTestSelezione.prototype.onTimer = function (timeNowMsec)
{
	if (this.timeStarted == 0)
		this.timeStarted = timeNowMsec;
	var timeElapsedMSec = timeNowMsec - this.timeStarted;
	
	//console.log ("TaskTestSelezione::onTimer => sel[" +this.selNum +"] attuatore[" +this.iAttuatore +"] fase[" +this.fase +"] cpu[" +this.cpuStatus +"]");
	if (this.iAttuatore == 12 || this.iAttuatore == 13 || this.iAttuatore == 14)
	{
		//questo è il caso del test "macinata" che prevede che prima si rimuova il gruppo, poi si macini, poi si rimetta il gruppo
		this.priv_handleTestMacina(timeElapsedMSec);
	}
	else
	{
		//nei test attuatori "normali", lascio fare il lavoro alla CPU e quando questa esce dallo stato 21, finisco pure io
		if (this.fase == 0)
		{
			this.fase = 1;
			
			//chiedo l'attivazione del motore
			//console.log ("TaskTestSelezione::ajax::testSelection");
			rhea.ajax ("testSelection", {"s":this.selNum, "d":this.iAttuatore} ).then( function(result)
			{
				if (result != "OK")
					pageSingleSelection_test_onFinish();
			})
			.catch( function(result)
			{
				pageSingleSelection_test_onFinish();
			});			
		}
		else
		{
			//aspetto almeno un paio di secondi
			if (timeElapsedMSec < 2000)
				return;
			//monitoro lo stato di cpu per capire quando esce da 21 e terminare
			if (this.cpuStatus != 21 && this.cpuStatus != 3 && this.cpuStatus != 101 && this.cpuStatus != 105) //21==eVMCState_TEST_ATTUATORE_SELEZIONE
				pageSingleSelection_test_onFinish();
		}
	}
}

TaskTestSelezione.prototype.onEvent_cpuStatus  = function(statusID, statusStr, flag16)	{ this.cpuStatus = statusID; pleaseWait_header_setTextL (statusStr); }
TaskTestSelezione.prototype.onEvent_cpuMessage = function(msg, importanceLevel)		{ rheaSetDivHTMLByName("footer_C", msg); pleaseWait_header_setTextR(msg); }
TaskTestSelezione.prototype.onFreeBtn1Clicked	= function(ev)
{ 
	if (this.iAttuatore != 12 && this.iAttuatore != 13 && this.iAttuatore != 14)
		return;
	
	switch (this.fase)
	{
	case 1:		pleaseWait_btn1_hide(); pleaseWait_btn2_hide(); this.fase = 2; break;
	case 21: 	pleaseWait_btn1_hide(); this.fase=30; break;
	case 41: 	pleaseWait_btn1_hide(); this.fase=50; break;
	}
}
TaskTestSelezione.prototype.onFreeBtn2Clicked	= function(ev)
{ 
	if (this.iAttuatore != 12 && this.iAttuatore != 13 && this.iAttuatore != 14)
		return;
	
	switch (this.fase)
	{
	case 1:		pleaseWait_btn1_hide(); pleaseWait_btn2_hide(); this.fase = 200; break;
	case 41:	pleaseWait_btn1_hide(); pleaseWait_btn2_hide(); this.fase = 2; break;
	}
}

TaskTestSelezione.prototype.onFreeBtnTrickClicked= function(ev)							{}

TaskTestSelezione.prototype.priv_handleTestMacina = function (timeElapsedMSec)
{
	var me = this;
	
	switch (this.fase)
	{
	case 0:
		me.fase = 1;
		pleaseWait_show();
		pleaseWait_calibration_show();
		pleaseWait_calibration_setText("Scoate grup cafea și apasă CONTINUARE"); //Please remove the brewer, then press CONTINUE
		pleaseWait_btn1_setText("CONTINUĂ");
		pleaseWait_btn1_show();
		pleaseWait_btn2_setText("ANULEAZĂ");
		pleaseWait_btn2_show();
		break;
		
	case 1:	//attendo btn CONTINUE / ABORT
		break;
		
	case 2: //verifico che il gruppo sia scollegato, altrimenti goto 0
		rhea.ajax ("getGroupState", "").then( function(result)
		{
			//console.log ("TaskCalibMotor, grpState[" +result +"]");
			if (result=="0")
				me.fase = 10;
			else
				me.fase = 0;
		})
		.catch( function(result)
		{
			me.fase = 0;
		});			
		me.fase = 3;
		break;
		
	case 3:	//attendo risposta CPU
		break;
		
	case 10:  //ok, il gruppo è scollegato, chiedo a CPU di attivare la macina
		me.fase = 11;
		pleaseWait_calibration_setText ("Râșniță în funcțiune"); //Grinder is running
		rhea.ajax ("testSelection", {"s":me.selNum, "d":me.iAttuatore} ).then( function(result)
		{
			if (result == "OK")
				me.fase = 20;
			else
				me.fase = 10;
		})
		.catch( function(result)
		{
			me.fase = 200;
		});					
	break;
		
	case 11: //attendo risposta di CPU
		break;
		
	//attendo la fine della macinata (ovvero quando la CPU passa in stato != 21)
	case 20:	me.fase = 21; break;
	case 21:	me.fase = 22; break;
	case 22:	me.fase = 23; break;
	case 23:
		if (me.cpuStatus != 21 && me.cpuStatus != 101) //21==eVMCState_TEST_ATTUATORE_SELEZIONE
			me.fase = 40;
		break;
	
	case 40: //chiedo di rimettere a posto il gruppo
		pleaseWait_calibration_setText("Repoziționează grup și apasă CONTINUARE sau apasă REPETĂ pentru a măcina din nou"); //Place the brewer into position then press CONTINUE, or press REPEAT to grind again
		pleaseWait_btn1_show();
		pleaseWait_btn2_setText("REPETĂ");
		pleaseWait_btn2_show();		
		me.fase = 41;
		break;
		
	case 41:
		break;
		
	case 50: //verifico che il gruppo sia collegato, altrimenti goto 40
		rhea.ajax ("getGroupState", "").then( function(result)
		{
			if (result=="1")
				me.fase = 200;
			else
				me.fase = 40;
		})
		.catch( function(result)
		{
			me.fase = 40;
		});			
		me.fase = 51;
		break;
		
	case 51://attendo risposta CPU
		break;
		
	case 200:
		me.fase = 201;
		pleaseWait_btn1_hide();
		pleaseWait_calibration_hide();
		pageSingleSelection_test_onFinish();
		break;
	}
	
}


/**********************************************************
 * TaskDevices
 */
function TaskDevices()
{
	this.what = 0;
	this.fase = 0;
	this.cpuStatus = 0;
	this.selNum = 0;
	this.enterQueryMacinePos(1);
}

TaskDevices.prototype.enterQueryMacinePos = function(macina_1o2)
{	
	this.what = 0;
	this.fase = 0;
	this.whichMacinaToQuery = macina_1o2;
}

TaskDevices.prototype.enterSetMacinaPos = function(macina_1o2, targetValue)
{	
	this.what = 1;
	this.fase = 0;
	this.macina = macina_1o2;
	pleaseWait_show();
	rhea.sendStartPosizionamentoMacina(macina_1o2, targetValue);
}

TaskDevices.prototype.runSelection = function(selNum)							{ this.what = 2; this.fase = 0; this.selNum = selNum; pleaseWait_show(); }
TaskDevices.prototype.runModemTest = function() 								{ this.what = 3; this.fase = 0; pleaseWait_show(); }
TaskDevices.prototype.messageBox = function (msg)
{
	this.whatBeforeMsgBox = this.what;
	this.what = 4;
	this.fase = 0;
	pleaseWait_show();
	pleaseWait_rotella_hide();
	pleaseWait_btn1_setText("OK");
	pleaseWait_btn1_show();
	pleaseWait_freeText_show();
	pleaseWait_freeText_setText(msg);
}

TaskDevices.prototype.snackEnterProg = function()									{ this.what = 9; this.fase = 0; pleaseWait_show(); }
TaskDevices.prototype.runTestAssorbGruppo = function()								{ this.what = 5; this.fase = 0; pleaseWait_show(); }
TaskDevices.prototype.runTestAssorbMotoriduttore = function()						{ this.what = 6; this.fase = 0; pleaseWait_show(); }
TaskDevices.prototype.runGrinderSpeedTest = function(macina1o2)						{ this.what = 7; this.fase = 0; this.macina1o2=macina1o2; this.speed1=0; this.speed2=0; this.gruppoTolto=0; pleaseWait_show(); }
TaskDevices.prototype.runScivoloBrewmatic = function (perc0_100)					{ this.what = 8; this.fase = 0; this.speed1=perc0_100; pleaseWait_show(); }
TaskDevices.prototype.onEvent_cpuStatus  = function(statusID, statusStr, flag16)	{ this.cpuStatus = statusID; pleaseWait_header_setTextL(statusStr); }
TaskDevices.prototype.onEvent_cpuMessage = function(msg, importanceLevel)			
{ 
	rheaSetDivHTMLByName("footer_C", msg); pleaseWait_header_setTextR(msg); 
	if (this.what == 9)
	{
		//siamo in snaxk prog
		pleaseWait_freeText_setText(msg);
		
	}
}
TaskDevices.prototype.onFreeBtn1Clicked	 = function(ev)
{
	switch (this.what)
	{
	case 1:
		//siamo in regolazione apertura vgrind
		if (this.fase > 0)
		{
			pleaseWait_btn1_hide();
			
			//Attivo la macina
			rhea.ajax ("runMotor", { "m":10+this.macina, "d":50, "n":1, "p":0}).then( function(result)
			{
				setTimeout ( function() {pleaseWait_btn1_show();}, 5000);
			})
			.catch( function(result)
			{
				pleaseWait_btn1_show();
			});								
		}
		break;
		
	case 4: //msgbox
		pleaseWait_hide();
		this.what = this.whatBeforeMsgBox;
		break;
		
	case 5: //test assorb gruppo
		this.fase = 90;	//ho premuto CLOSE al termine del test
		break;

	case 6: //test assorb motoriduttore
		switch (this.fase)
		{
			case 1:	//ho premuto CONTINUE nella fase di "prego rimuovere il gruppo"
				this.fase = 2;
				pleaseWait_btn1_hide(); pleaseWait_btn2_hide();
				break;
				
			case 80: //ho premuto CONTINUE al termine del test
				this.fase = 85;		
				pleaseWait_btn1_hide(); pleaseWait_btn2_hide();
				break;
				
			case 86: //ho premuto CONTINUE nella fase di "prego rimettere a posto il gruppo"
				this.fase = 87;
				pleaseWait_btn1_hide(); pleaseWait_btn2_hide();
				break;
				
		}
		break;
		
	case 7: //grinder speed Test
		switch (this.fase)
		{
			case 1:		this.fase = 2; 	pleaseWait_btn1_hide(); pleaseWait_btn2_hide(); break; 	//ho premuto CONTINUE
			case 6:		this.fase = 10; pleaseWait_btn1_hide(); pleaseWait_btn2_hide(); break; 	//ho premuto CONTINUE
			case 21:	this.fase = 30; pleaseWait_btn1_hide(); pleaseWait_btn2_hide(); break;	//ho premuto CONTINUE
			case 41:	this.fase = 50; pleaseWait_btn1_hide(); pleaseWait_btn2_hide(); break;	//ho premuto CONTINUE
			case 61:	this.fase = 70; pleaseWait_btn1_hide(); pleaseWait_btn2_hide(); break;	//ho premuto CLOSE
			case 71:	this.fase = 72; pleaseWait_btn1_hide(); pleaseWait_btn2_hide(); break;	//ho premuto CONTINUE
		}
		break;
		
	case 9: //snack prog (Exit)
		this.what = 0;
		pleaseWait_hide();
		rhea.ajax ("snackExitProg", "").then( function(result)
		{
		})
		.catch( function(result)
		{
		});		
		break;
	}
}

TaskDevices.prototype.onFreeBtn2Clicked	 = function(ev)
{
	switch (this.what)
	{
	case 5: //test assorb gruppo
		switch (this.fase)
		{
			case 80: //ho premuto REPEAT nella fase finale del test
				this.fase = 0;
				pleaseWait_btn1_hide(); pleaseWait_btn2_hide();
				break;		
		}
		break;
		
	case 6: //test assorb motoriduttore
		switch (this.fase)
		{
			case 1:	//ho premuto ABORT nella fase di "prego rimuovere il gruppo"
				this.fase = 90;
				pleaseWait_btn1_hide(); pleaseWait_btn2_hide();
				break;
				
			case 80: //ho premuto REPEAT nella fase finale del test
				this.fase = 10;
				pleaseWait_btn1_hide(); pleaseWait_btn2_hide();
				break;
				
		}
		break;		
		
	case 7: //grinder speed Test
		this.fase = 70;
		pleaseWait_btn1_hide(); pleaseWait_btn2_hide();
		break;
		
	case 8: //test scivolo Brewmatic
		pleaseWait_btn1_hide(); pleaseWait_btn2_hide();
		this.fase = 90;
		break;
	}
}

TaskDevices.prototype.onFreeBtnTrickClicked= function(ev)						{}


TaskDevices.prototype.onTimer = function (timeNowMsec)
{
	if (this.what == 0)
		this.priv_handleRichiestaPosizioneMacina();
	else if (this.what == 1)
		this.priv_handleRegolazionePosizioneMacina();
	else if (this.what == 2)
		this.priv_handleRunSelection(timeNowMsec);
	else if (this.what == 3)
		this.priv_handleModemTest(timeNowMsec);
	else if (this.what == 5)
		this.priv_handleTestAssorbGruppo(timeNowMsec);
	else if (this.what == 6)
		this.priv_handleTestAssorbMotoriduttore(timeNowMsec);
	else if (this.what == 7)
		this.priv_handleGrinderSpeedTest(timeNowMsec);
	else if (this.what == 8)
		this.priv_handleTestScivoloBrewmatic(timeNowMsec);
	else if (this.what == 9)
		this.priv_handleSnackProg (timeNowMsec);
}

TaskDevices.prototype.priv_handleSnackProg = function(timeNowMsec)
{
	switch (this.fase)
	{
	case 0:
		this.fase = 10;
		pleaseWait_rotella_hide();
		pleaseWait_btn1_setText("IEȘI DIN PROGRAMARE");
		pleaseWait_btn1_show();
		pleaseWait_freeText_setText("");
		pleaseWait_freeText_show();
		break;
		
	case 10:
		break;
	}
		
}

TaskDevices.prototype.priv_queryCupCoverSensorLiveValue = function(timeNowMsec)
{
	
}

TaskDevices.prototype.priv_handleRunSelection = function(timeNowMsec)
{
	if (this.fase == 0)
	{
		this.fase = 1;
		this.timeStartedMSec = timeNowMsec;
		
		rhea.ajax ("testSelection", {"s":this.selNum, "d":0} ).then( function(result)
		{
			if (result != "OK")
			{
				this.fase = 2;
				pageDevices_vgrind_runSelection_onFinish();
			}
		})
		.catch( function(result)
		{
			this.fase = 2;
			pageDevices_vgrind_runSelection_onFinish();
		});			
	}
	else if (this.fase == 1)
	{
		//aspetto almeno un paio di secondi
		if ((timeNowMsec - this.timeStartedMSec) < 2000)
			return;
		//monitoro lo stato di cpu per capire quando esce da 3 (prep bevanda)
		if (this.cpuStatus != 3 && this.cpuStatus != 101 && this.cpuStatus != 105)
		{
			this.fase = 2;
			pageDevices_vgrind_runSelection_onFinish();
		}
	}
	
}

TaskDevices.prototype.priv_handleRichiestaPosizioneMacina = function()
{
	//aggiornamento lettura sul sensore "cup"
	rhea.ajax ("getCupSensorLiveValue", "" ).then( function(result)
	{
		rheaSetDivHTMLByName("divCupSensorLiveValue", result);
		
		//nel caso di sensore "brewmatic"
		switch (parseInt(result))
		{
		case 0:rheaSetDivHTMLByName("divCupSensorLiveValueBrewmatic", "(173+) mm"); break;
		case 1:rheaSetDivHTMLByName("divCupSensorLiveValueBrewmatic", "(145-173) mm"); break;
		case 2:rheaSetDivHTMLByName("divCupSensorLiveValueBrewmatic", "(116-145) mm"); break;
		case 3:rheaSetDivHTMLByName("divCupSensorLiveValueBrewmatic", "(101-116) mm"); break;
		case 4:rheaSetDivHTMLByName("divCupSensorLiveValueBrewmatic", "(90-101) mm"); break;
		case 5:rheaSetDivHTMLByName("divCupSensorLiveValueBrewmatic", "(65-90) mm"); break;
		default: rheaSetDivHTMLByName("divCupSensorLiveValueBrewmatic", "NICI UN PAHAR DETECTAT"); break;
		}
		
		//nel caso di sensore Wenglor
		switch (parseInt(result))
		{
		default: rheaSetDivHTMLByName("divCupSensorLiveValueWenglor", "NICI UN PAHAR DETECTAT"); break;
		case 1:rheaSetDivHTMLByName("divCupSensorLiveValueWenglor", "PAHAR MIC (jos)"); break;
		case 2:rheaSetDivHTMLByName("divCupSensorLiveValueWenglor", "PAHAR MIC (sus)"); break;
		case 3:rheaSetDivHTMLByName("divCupSensorLiveValueWenglor", "PAHAR MARE"); break;
		}		
	})
	.catch( function(result)
	{
	});
	
	
	//aggiornamento posizione macine
	if (da3.isGruppoMicro())
		return;	
	var me = this;
	if (this.fase == 0)
	{
		//chiede la posizione della macina
		this.fase = 1;
		rhea.ajax ("getPosMacina", {"m":this.whichMacinaToQuery}).then( function(result)
		{
			var obj = JSON.parse(result);
			var macina_1to4 = obj.m;
			rheaSetDivHTMLByName("pageDevices_vg" +macina_1to4, obj.v);
			me.fase = 0;
		})
		.catch( function(result)
		{
			me.fase = 0;
		});			
		return;
	}
	else
	{
		//aspetto una risposta alla query precedente
	}

}

TaskDevices.prototype.priv_handleRegolazionePosizioneMacina = function()
{
	switch (this.fase)
	{
		case 0: 
			this.fase=1; 
			
			pleaseWait_freeText_setText("Te rog așteaptă ca varigrind să își ajusteze poziția"); //Please wait while the varigrind is adjusting its position
			pleaseWait_freeText_show();
			/*pleaseWait_freeText_setText("While the varigrind is opening/closing, you can press RUN GRINDER to run the grinder in order to facilitate the operation.");
			pleaseWait_freeText_show();
			pleaseWait_btn1_setText("RUN GRINDER");
			pleaseWait_btn1_show();
			*/
			break;
		case 1: this.fase=2; break;
		case 2: 
			this.priv_queryMacina(this.macina);
			this.fase=3; 
			break;
		
		case 3: 
			//a questo punto CPU dovrebbe essere in stato 102 e dovrebbe rimanerci fino a fine operazione
			if (this.cpuStatus != 102 && this.cpuStatus != 101)
			{
				this.enterQueryMacinePos(this.macina);
				pleaseWait_hide();
				return;
			}
			
			this.fase = 2;
			break;
	}
}

TaskDevices.prototype.priv_queryMacina = function(macina_1o2)
{
	if (da3.isGruppoMicro())
		return;	
	rhea.ajax ("getPosMacina", {"m":macina_1o2}).then( function(result)
	{
		var obj = JSON.parse(result);
		rheaSetDivHTMLByName("pageDevices_vg" +macina_1o2, obj.v);
		pleaseWait_freeText_setText("Te rog așteaptă ca varigrind să își ajusteze poziția" +"  [" +obj.v +"]"); //Please wait while the varigrind is adjusting its position [current_value]
	})
	.catch( function(result)
	{
	});			
}


TaskDevices.prototype.priv_handleModemTest = function(timeNowMsec)
{
	var me = this;
	switch (me.fase)
	{
	case 0:
		pleaseWait_freeText_setText ("Modem test is starting...");
		pleaseWait_freeText_show();
		me.fase = 1;
		rhea.ajax ("startModemTest", "" ).then( function(result)
		{
			if (result == "OK")
				me.fase = 10;
			else
				me.fase = 90;
		})
		.catch( function(result)
		{
			me.fase = 90;
		});			
		break;


	case 1:
		//sono in attesa della risposta al comando "startModemTest"
		break;
		
	case 10:
		//ho ricevuto l'OK dal comando startModemTest. La CPU dovrebbe andare in stato 22 e rimanerci fino alla fine
		//della procedura di test
		//Aspetto un paio di secondi per dare tempo alla CPU di cambiare di stato
		pleaseWait_freeText_setText ("Modem test is running, please wait...");
		me.timeStartedMSec = timeNowMsec;
		me.fase = 11;
		break;
		
	case 11:
		//aspetto un paio di secondi
		if ((timeNowMsec - me.timeStartedMSec) >= 2000)
			me.fase = 20;
		break;
		
	case 20:
		//monitoro lo stato di cpu per capire quando questa esce da 22 (test_modem)
		if (me.cpuStatus != 22 && me.cpuStatus != 101 && me.cpuStatus != 105)
		{
			pleaseWait_freeText_setText ("Modem test finished");
			me.fase = 90;
		}
		break;
		
	case 90: //fine
		pleaseWait_hide();
		me.what = 0;
		break;
	}
	
}


TaskDevices.prototype.priv_handleTestAssorbGruppo = function(timeNowMsec)
{
	var me = this;
	switch (me.fase)
	{
	case 0:
		pleaseWait_freeText_setText ("Testul începe...");
		pleaseWait_freeText_show();
		pleaseWait_rotella_show();
		me.fase = 1;
		me.test_fase = 0;
		rhea.ajax ("startTestAssGrp", "" ).then( function(result)
		{
			if (result == "OK")
				me.fase = 10;
			else
				me.fase = 90;
		})
		.catch( function(result)
		{
			me.fase = 90;
		});			
		break;


	case 1:
		//sono in attesa della risposta al comando "start Test"
		break;
		
	case 10:
		//ho ricevuto l'OK dal comando start Test. Da ora in poi, pollo lo stato del test fino a che non finisce
		pleaseWait_freeText_setText ("Testul este în curs, te rog așteaptă...<br>Faza actuală: " +me.test_fase +"/5");
		me.fase = 11;
		break;
		
	case 11:
		//query stato del test
		rhea.ajax ("getStatTestAssGrp", "" ).then( function(result)
		{
			var obj = JSON.parse(result);
			me.test_fase= obj.fase;
			
			if (obj.esito != 0)
			{
				//errore
				pleaseWait_freeText_setText ("Testul este finalizat.<br>Result: FAILED<br><br>");
				pleaseWait_btn1_setText("ÎNCHIDE");
				pleaseWait_btn1_show();	
				pleaseWait_btn2_setText("REPETĂ");
				pleaseWait_btn2_show();	
				me.fase = 80;
			}
			else
			{
				if (obj.fase != 5)
					me.fase = 10;			
				else
				{
					//test terminato con successo
					var html = "<table class='dataAudit'>"
								+"<tr><td>&nbsp;</td><td align='center'><b>ASCENT</b></td><td align='center'><b>DESCENT</b></td></tr>"
								+"<tr><td>Medium absorption</td><td align='center'>" +obj.r1up +"</td><td align='center'>" +obj.r1down +"</td></tr>"
								+"<tr><td>Maximum absorption</td><td align='center'>" +obj.r2up +"</td><td align='center'>" +obj.r2down +"</td></tr>"
								+"<tr><td>Time</td><td align='center'>" +obj.r3up +"</td><td align='center'>" +obj.r3down +"</td></tr>"
								+"<tr><td>Medium absorption during cycle 1</td><td align='center'>" +obj.r4up +"</td><td align='center'>" +obj.r4down +"</td></tr>"
								+"<tr><td>Medium absorption during cycle 2</td><td align='center'>" +obj.r5up +"</td><td align='center'>" +obj.r5down +"</td></tr>"
								+"<tr><td>Medium absorption during cycle 3</td><td align='center'>" +obj.r6up +"</td><td align='center'>" +obj.r6down +"</td></tr>"
								+"</table>";
					pleaseWait_freeText_setText ("Testul este finalizat.<br>Results:<br><br>" +html);		
					pleaseWait_btn1_setText("ÎNCHIDE");
					pleaseWait_btn1_show();	
					pleaseWait_btn2_setText("REPETĂ");
					pleaseWait_btn2_show();	
					me.fase = 80;
				}
			}
		})
		.catch( function(result)
		{
			me.fase = 10;
		});			
		break;
		
	case 80: //attendo pressione di un tasto per finire
		pleaseWait_rotella_hide();
		break;
		
	case 90: //fine
		pleaseWait_hide();
		me.what = 0;
		break;
	}	
}

TaskDevices.prototype.priv_handleTestAssorbMotoriduttore = function(timeNowMsec)
{
	var me = this;
	switch (me.fase)
	{
	case 0: //prego rimuovere il gruppo
		me.fase = 1;
		pleaseWait_show();
		pleaseWait_freeText_setText ("Scoate grup cafea și apasă CONTINUARE");
		pleaseWait_freeText_show();
		pleaseWait_btn1_setText("CONTINUĂ");
		pleaseWait_btn1_show();
		pleaseWait_btn2_setText("ANULEAZĂ");
		pleaseWait_btn2_show();	
		break;
		
	case 1:	//attendo btn CONTINUE
		break;
		
	case 2: //verifico che il gruppo sia scollegato, altrimenti goto 0
		rhea.ajax ("getGroupState", "").then( function(result)
		{
			if (result=="0")
				me.fase = 10;
			else
				me.fase = 0;
		})
		.catch( function(result)
		{
			me.fase = 0;
		});			
		me.fase = 3;
		break;
		
	case 3:	//attendo risposta CPU
		break;
		
		
		
	case 10: //inizio del test
		pleaseWait_freeText_setText ("Testul începe ...");
		pleaseWait_rotella_show();
		me.fase = 11;
		me.test_fase = 0;
		rhea.ajax ("startTestAssMotorid", "" ).then( function(result)
		{
			if (result == "OK")
				me.fase = 20;
			else
				me.fase = 80;
		})
		.catch( function(result)
		{
			me.fase = 80;
		});			
		break;


	case 11:
		//sono in attesa della risposta al comando "start Test"
		break;
		
	case 20:
		//ho ricevuto l'OK dal comando start Test. Da ora in poi, pollo lo stato del test fino a che non finisce
		pleaseWait_freeText_setText ("Testul este în curs, te rog așteaptă...<br>Faza actuală: " +me.test_fase +"/4");
		me.fase = 21;
		break;
		
	case 21:
		//query stato del test
		rhea.ajax ("getStatTestAssMotorid", "" ).then( function(result)
		{
			var obj = JSON.parse(result);
			me.test_fase= obj.fase;
			
			if (obj.esito != 0)
			{
				//errore
				pleaseWait_freeText_setText ("Testul este finalizat.<br>Result: FAILED<br><br>");
				pleaseWait_btn1_setText("ÎNCHIDE");
				pleaseWait_btn1_show();	
				pleaseWait_btn2_setText("REPETĂ");
				pleaseWait_btn2_show();	
				me.fase = 80;
			}
			else
			{
				if (obj.fase >= 4)
				{
					//test terminato con successo
					var html = "<table class='dataAudit'>"
								+"<tr><td>&nbsp;</td><td align='center'><b>ASCENT</b></td><td align='center'><b>DESCENT</b></td></tr>"
								+"<tr><td>Medium absorption</td><td align='center'>" +obj.r1up +"</td><td align='center'>" +obj.r1down +"</td></tr>"
								+"</table>";
					pleaseWait_freeText_setText ("Testul este finalizat.<br>Results:<br><br>" +html);		
					pleaseWait_btn1_setText("CONTINUĂ");
					pleaseWait_btn1_show();	
					pleaseWait_btn2_setText("REPETĂ");
					pleaseWait_btn2_show();	
					me.fase = 80;
				}
				else
					me.fase = 20;			

			}
		})
		.catch( function(result)
		{
			me.fase = 20;
		});			
		break;
		
	case 80: //attendo pressione di un tasto per proseguire
		pleaseWait_rotella_hide();
		break;
		
	case 85: //prego rimettere a posto il gruppo
		pleaseWait_freeText_setText("Repoziționează grup cafea și apasă CONTINUARE"); //Place the brewer into position, then press CONTINUE
		pleaseWait_btn1_setText("CONTINUĂ");
		pleaseWait_btn1_show();
		pleaseWait_rotella_show();
		me.fase = 86;
		break;
		
	case 86:	//attendo btn CONTINUE
		break;
		
	case 87: //verifico che il gruppo sia collegato, altrimenti goto 85
		rhea.ajax ("getGroupState", "").then( function(result)
		{
			if (result=="1")
				me.fase = 90;
			else
				me.fase = 85;
		})
		.catch( function(result)
		{
			me.fase = 85;
		});			
		me.fase = 88;
		break;
		
	case 88:	//attendo risposta CPU
		break;		
		
	
		
	case 90: //fine
		pleaseWait_hide();
		me.what = 0;
		break;
	}	
}


TaskDevices.prototype.priv_handleGrinderSpeedTest = function(timeNowMsec)
{
	var DURATA_MACINATA_SEC = 10;
	var me = this;
	
	//console.log ("fase[" +me.fase +"]");	
	switch (me.fase)
	{
	case 0:
		me.fase = 1;
		pleaseWait_show();
		pleaseWait_rotella_hide();
		pleaseWait_freeText_setText ("<b>VITEZĂ RÂȘNIȚĂ PENTRU OFF09</b><br><br>Scoate grup cafea și apasă CONTINUARE"); //Please remove the brewer, then press CONTINUE
		pleaseWait_freeText_show();
		pleaseWait_btn1_setText("CONTINUĂ");
		pleaseWait_btn1_show();
		pleaseWait_btn2_setText("ANULEAZĂ");
		pleaseWait_btn2_show();	
		break;
		
	case 1:	//attendo btn CONTINUE
		break;
		
	case 2: //verifico che il gruppo sia scollegato, altrimenti goto 0
		pleaseWait_rotella_show();
		rhea.ajax ("getGroupState", "").then( function(result)
		{
			if (result=="0")
			{
				me.gruppoTolto=1;
				me.fase = 5;
			}
			else
				me.fase = 0;
		})
		.catch( function(result)
		{
			me.fase = 0;
		});			
		me.fase = 3;
		break;
		
	case 3:	//attendo risposta CPU
		break;

	
	//************************************************************ STEP 1	
	case 5: //step 1
		me.fase = 6;
		pleaseWait_show();
		pleaseWait_rotella_hide();
		pleaseWait_freeText_setText ("<b>VITEZĂ RÂȘNIȚĂ PENTRU OFF09</b><br><br>PASUL 1<br>Te rog <b>închide clopotul de cafea</b> (maneta portocalie) și apasă CONTINUARE. Râșnița va funcționa circa 10 secunde pentru a se goli.");
		pleaseWait_freeText_show();
		pleaseWait_btn1_setText("CONTINUĂ");
		pleaseWait_btn1_show();
		pleaseWait_btn2_setText("ANULEAZĂ");
		pleaseWait_btn2_show();	
		break;
		
	case 6: //attendo CONTINUE o ABORT
		break;
		
	case 10: //macino per una 10a di secondi in modo da svuotare la macina
		pleaseWait_rotella_show();
		me.fase = 11;
		rhea.ajax ("startGrnSpeedTest", { "m":this.macina1o2, "d":DURATA_MACINATA_SEC}).then( function(result)
			{
				if (result == "OK")
					me.fase = 12;
				else
					me.fase = 5;
				
			})
			.catch( function(result)
			{
				me.fase = 5;
			});
		break;		
		
	case 11: //attendo risposta CPU
		break;	
		
	case 12: //attendo fine macinata. Prima verifico di leggere this.cpuStatus==eVMCState_GRINDER_SPEED_TEST(106) e poi attendo che this.cpuStatus torni eVMCState_DISPONIBILE
		if (this.cpuStatus == 106)
			me.fase = 13;
		break;
		
	case 13:
		if (this.cpuStatus != 106)
			me.fase = 20;
		break;
	
	//************************************************************ STEP 2	
	case 20:
		me.fase = 21;
		pleaseWait_rotella_hide();
		pleaseWait_freeText_setText ("<b>VITEZĂ RÂȘNIȚĂ PENTRU OFF09</b><br><br>PASUL 2<br>Cu râșnița goală, putem testa pentru a vedea valoarea medie raportată de senzor când râșnița este goală. Apasă CONTINUARE pentru a continua. Râșnița va funcționa circa 10 secunde.");		
		pleaseWait_freeText_show();
		pleaseWait_btn1_setText("CONTINUĂ");
		pleaseWait_btn1_show();
		pleaseWait_btn2_setText("ANULEAZĂ");
		pleaseWait_btn2_show();
		break;
		
	case 21: //attendo CONTINUE o ABORT
		break;		
	
	case 30: //macino per una 10a di secondi in modo da svuotare la macina
		me.fase = 31;
		pleaseWait_rotella_show();
		rhea.ajax ("startGrnSpeedTest", { "m":this.macina1o2, "d":DURATA_MACINATA_SEC}).then( function(result)
			{
				if (result == "OK")
					me.fase = 32;
				else
					me.fase = 20;
				
			})
			.catch( function(result)
			{
				me.fase = 20;
			});
		break;
	
	case 31: //attendo risposta CPU	
		break;
		
	case 32: //attendo fine macinata. Prima verifico di leggere this.cpuStatus==eVMCState_GRINDER_SPEED_TEST(106) e poi attendo che this.cpuStatus torni eVMCState_DISPONIBILE
		if (this.cpuStatus == 106)
			me.fase = 33;
		break;
		
	case 33:
		if (this.cpuStatus != 106)
			me.fase = 34;
		break;
		
	case 34: //la macinata è finita, chiedo la speed calcolata
		me.fase = 35;
		rhea.ajax ("getLastGrndSpeedValue", "").then( function(result)
		{
			me.speed1 = parseInt(result);
			me.fase = 40;
			console.log ("SPEED1: " +me.speed1);
		})
		.catch( function(result)
		{
			me.fase = 20;
		});
		break;

	case 35: //attendo risposta CPU	
		break;


	//************************************************************ STEP 3
	case 40:
		me.fase = 41;
		pleaseWait_rotella_hide();
		pleaseWait_freeText_setText ("<b>VITEZĂ RÂȘNIȚĂ PENTRU OFF09</b><br><br>PASUL 3<br>Acum <b>deschide clopotul de cafea</b> (maneta portocalie) și apoi apasă CONTINUARE pentru a continua. Râșnița va funcționa circa 10 secunde.");
		pleaseWait_freeText_show();
		pleaseWait_btn1_setText("CONTINUĂ");
		pleaseWait_btn1_show();
		pleaseWait_btn2_setText("ANULEAZĂ");
		pleaseWait_btn2_show();
		break;
		
	case 41: //attendo CONTINUE o ABORT
		break;		

	case 50: //macino per una 10a di secondi in modo da svuotare la macina
		me.fase = 51;
		pleaseWait_rotella_show();
		rhea.ajax ("startGrnSpeedTest", { "m":this.macina1o2, "d":DURATA_MACINATA_SEC}).then( function(result)
			{
				if (result == "OK")
					me.fase = 52;
				else
					me.fase = 40;
				
			})
			.catch( function(result)
			{
				me.fase = 40;
			});
		break;
		
	case 51: //attendo risposta CPU	
		break;
		
	case 52: //attendo fine macinata. Prima verifico di leggere this.cpuStatus==eVMCState_GRINDER_SPEED_TEST(106) e poi attendo che this.cpuStatus torni eVMCState_DISPONIBILE
		if (this.cpuStatus == 106)
			me.fase = 53;
		break;
		
	case 53:
		if (this.cpuStatus != 106)
			me.fase = 54;
		break;
		
	case 54: //la macinata è finita, chiedo la speed calcolata
		me.fase = 55;
		rhea.ajax ("getLastGrndSpeedValue", "").then( function(result)
		{
			me.speed2 = parseInt(result);
			me.fase = 60;
			console.log ("SPEED1: " +me.speed2);
		})
		.catch( function(result)
		{
			me.fase = 40;
		});
		break;
		
	case 55: //attendo risposta CPU	
		break;
		
		
	//************************************************************ STEP 4
	case 60:
		me.fase = 61;
		pleaseWait_rotella_hide();
		pleaseWait_freeText_setText ("<b>VITEZĂ RÂȘNIȚĂ PENTRU OFF09</b><br><br>REZULTAT:<br>Viteză medie când râșnița este goală: <b>" +me.speed1 +"</b><br>Viteză medie când râșnița nu este goală: <b>" +me.speed2 +"</b><br>");		
		pleaseWait_freeText_show();
		pleaseWait_btn1_setText("ÎNCHIDE");
		pleaseWait_btn1_show();
		break;

	case 61: //attendo CLOSE
		break;		

		
	//************************************************************
	case 70: //chiedo di rimettere a posto il gruppo
		if (me.gruppoTolto == 0)
		{
			me.fase = 90;
		}
		else
		{
			me.fase = 71;
			pleaseWait_rotella_hide();
			pleaseWait_freeText_setText("<b>VITEZĂ RÂȘNIȚĂ PENTRU OFF09</b><br><br>Repoziționează grup cafea și apasă CONTINUARE"); //Place the brewer into position, then press CONTINUE
			pleaseWait_freeText_show();
			pleaseWait_btn1_setText("CONTINUĂ");
			pleaseWait_btn1_show();
		}
		break;
		
	case 71: //aspetto continue
		break;
		
	case 72: //verifico che il gruppo sia collegato, altrimenti goto 40
		me.fase = 73;
		pleaseWait_rotella_show();
		rhea.ajax ("getGroupState", "").then( function(result)
		{
			if (result=="1")
			{
				me.gruppoTolto = 0;
				me.fase = 90;
			}
			else
				me.fase = 70;
		})
		.catch( function(result)
		{
			me.fase = 70;
		});			
		
		break;
		
	case 73://attendo risposta CPU
		break;
		
	
	//************************************************************ FINE
	case 90: //fine
		pleaseWait_hide();
		me.what = 0;
		break;
	}
}

TaskDevices.prototype.priv_handleTestScivoloBrewmatic = function(timeNowMsec)
{
	switch (this.fase)
	{
	case 0:
		this.fase = 1;
		pleaseWait_show();
		pleaseWait_rotella_hide();
		pleaseWait_btn2_setText("STOP");
		pleaseWait_btn2_show();	
		
		rhea.ajax ("scivoloBrewmatic", { "perc": this.speed1 }).then( function(result)
		{
		})
		.catch( function(result)
		{
		});			
		break;
		
	case 1:	//attendo btn STOP
		break;
		
	//************************************************************ FINE
	case 90: //fine
		rhea.ajax ("scivoloBrewmatic", { "perc": 0 }).then( function(result)
		{
		})
		.catch( function(result)
		{
		});		
		
		pleaseWait_hide();
		this.what = 0;
		break;
	}
}

/**********************************************************
 * TaskDisintall
 */
function TaskDisintall()
{
	this.what = 0;
	this.fase = 0;
	this.cpuStatus = 0;
	this.isSecondUninstall = 0;
	this.sleepUntil_ms = 0;
}

TaskDisintall.prototype.onEvent_cpuStatus  = function(statusID, statusStr, flag16)	{ this.cpuStatus = statusID; pleaseWait_header_setTextL (statusStr); }
TaskDisintall.prototype.onEvent_cpuMessage = function(msg, importanceLevel)		{ rheaSetDivHTMLByName("footer_C", msg); pleaseWait_header_setTextR(msg); }

TaskDisintall.prototype.onFreeBtn1Clicked	 = function(ev)						
{
	switch (this.fase)
	{
		case 1:  pleaseWait_btn1_hide(); pleaseWait_btn2_hide(); this.fase=10; break;
		case 11: pleaseWait_btn1_hide(); pleaseWait_btn2_hide(); this.fase=20; break;
		case 21: pleaseWait_btn1_hide(); pleaseWait_btn2_hide(); this.fase=30; break;
		case 33: pleaseWait_btn1_hide(); pleaseWait_btn2_hide(); this.fase=35; rhea.sendButtonPress(10); break;
		
	}
}
TaskDisintall.prototype.onFreeBtn2Clicked	 = function(ev)						
{
	pleaseWait_btn1_hide(); 
	pleaseWait_btn2_hide();
	this.fase = 99;
}

TaskDisintall.prototype.onFreeBtnTrickClicked= function(ev)						{}

TaskDisintall.prototype.onTimer = function (timeNowMsec)
{
	var bBollitore400cc = 0;
	if (!da3.isInduzione())
	{
		if (!da3.isInstant())
		{
			if (!da3.hasCapab_BOLLITORE_BELTRAMI_650())
			{
				if (da3.read8(7072)==0)
					bBollitore400cc=1;
			}
		}
	}
	
	//console.log ("TaskDisintall, fase=" +this.fase);
	if (this.timeStarted == 0)
		this.timeStarted = timeNowMsec;
	var timeElapsedMSec = timeNowMsec - this.timeStarted;
	
	switch (this.fase)
	{
	case 0:
		this.fase = 1;
		pleaseWait_show();
		pleaseWait_freeText_show();
		pleaseWait_freeText_setText("DEZINSTALARE<br><br>Tava de picurare este goală?"); //DISINTALLATION<br><br>Is driptray empty?
		pleaseWait_btn1_setText("DA - CONTINUARE");
		pleaseWait_btn2_setText("ANULEAZĂ");
		pleaseWait_btn1_show();
		pleaseWait_btn2_show();
		break;
		
	case 1:
		break;
		
	case 10:
		this.fase = 11;
		pleaseWait_freeText_setText("DEZINSTALARE<br><br>Golește zaț cafea și apasă CONTINUARE"); //DISINTALLATION<br><br>Please remove coffee grounds, then press CONTINUE
		pleaseWait_btn1_setText("CONTINUĂ");
		pleaseWait_btn2_setText("ANULEAZĂ");
		pleaseWait_btn1_show();
		pleaseWait_btn2_show();
		break;
		
	case 11:
		break;
		
	case 20:
		this.fase = 21;
		pleaseWait_freeText_setText("DEZINSTALARE<br><br>Apasă START DEZINSTALARE pentru a continua, ANULEAZĂ pentru a anula operațiunea"); //DISINTALLATION<br><br>Press START DISINSTALLATION to continue, ABORT to cancel the operation
		pleaseWait_btn1_setText("START DEZINSTALARE");
		pleaseWait_btn2_setText("ANULEAZĂ");
		pleaseWait_btn1_show();
		pleaseWait_btn2_show();
		break;
		
	case 21:
		break;
		
	case 30:
		this.fase = 31;
		pleaseWait_show();
		pleaseWait_freeText_show();
		if (this.isSecondUninstall)
			pleaseWait_freeText_setText("DEZINSTALARE în curs, te rog așteaptă ...<br><br><br>Apasă butonul PROG pentru a anula"); //DISINTALLATION is running, please wait
		else
			pleaseWait_freeText_setText("DEZINSTALARE în curs, te rog așteaptă ..."); //DISINTALLATION is running, please wait
		rhea.sendStartDisintallation();
		break;
		
	case 31: 
		this.fase = 32;
		break;
		
	case 32:
		if (bBollitore400cc)
		{
			//devo aspettare qualche sec prima di mostrare i btn, altrimenti si rischia che CPU non senta il comando
			this.fase = 50;
			this.sleepUntil_ms = timeNowMsec + 2500;
		}		
		else
			this.fase = 36; 
		break;
		
	case 33:
		break;
		
	case 35:
		this.fase = 36;
		if (this.isSecondUninstall)
			pleaseWait_freeText_setText("DEZINSTALARE în curs, te rog așteaptă ...<br><br><br>Apasă butonul PROG pentru a anula"); //DISINTALLATION is running, please wait
		else
			pleaseWait_freeText_setText("DEZINSTALARE în curs, te rog așteaptă ..."); //DISINTALLATION is running, please wait
		break;
		
	
	case 36: this.fase = 37; break;
	case 37: this.fase = 38; break;
	case 38: 
		//a questo punto CPU dovrebbe già essere in stato eVMCState_DISINSTALLAZIONE (13)
		//quando ha finito, finisco pure io
		if (this.cpuStatus != 13 && this.cpuStatus != 101)
			this.fase = 40;
		break;
		
	case 40:
		var msgToOutput = "DEZINSTALARE terminată, OPREȘTE aparatul."; //DISINTALLATION finished, please shut down the machine
		if (bBollitore400cc)
			msgToOutput = "DEZINSTALARE terminată, ÎNCHIDE robinet boiler și OPREȘTE aparatul"; //EINSTALLATION finished, please CLOSE the boiler tap and SHUT DOWN the machine
		pleaseWait_freeText_setText(msgToOutput); //DISINTALLATION finished, please shut down the machine
		this.fase = 41;
		break;
		
	case 41:
		break;
		
	case 50:
		//aspetto senza fare niente per qualche secondo
		if (timeNowMsec >= this.sleepUntil_ms)
			this.fase = 51;
		break;
		
	case 51:
		if (this.isSecondUninstall)
			pleaseWait_freeText_setText("DEZINSTALARE<br><br>Deschide robinet boiler și apasă CONTINUARE ...<br><br><br>Apasă butonul PROG pentru a anula"); //DISINSTALLATION<br><br>Open boiler tap then press CONTINUE
		else
			pleaseWait_freeText_setText("DEZINSTALARE<br><br>Deschide robinet boiler și apasă CONTINUARE ..."); //DISINSTALLATION<br><br>Open boiler tap then press CONTINUE
		
		pleaseWait_btn1_setText("CONTINUĂ");
		pleaseWait_btn1_show();
		this.fase = 33;
		break;
		
	case 99:
		pleaseWait_hide();
		pageMaintenance_disinstall_onAbort();
		break;
	}
}

/**********************************************************
 * TaskDataAudit
 */
function TaskDataAudit()
{
	this.what = 0;
	this.fase = 0;
	this.cpuStatus = 0;
	this.fileID = 0;
	this.buffer = null;
	this.bufferSize = 0;
}

TaskDataAudit.prototype.onEvent_cpuStatus  = function(statusID, statusStr, flag16)	{ this.cpuStatus = statusID; pleaseWait_header_setTextL (statusStr); }
TaskDataAudit.prototype.onEvent_cpuMessage = function(msg, importanceLevel)		{ rheaSetDivHTMLByName("footer_C", msg); pleaseWait_header_setTextR(msg); }

TaskDataAudit.prototype.onFreeBtn1Clicked	 = function(ev)						
{
	switch (this.fase)
	{
		case 202:	pleaseWait_btn1_hide(); pleaseWait_btn2_hide(); this.fase = 210; break;
	}
}

TaskDataAudit.prototype.onFreeBtn2Clicked	 = function(ev)						{}
TaskDataAudit.prototype.onFreeBtnTrickClicked= function(ev)						{}


TaskDataAudit.prototype.onTimer = function (timeNowMsec)
{
	if (this.timeStarted == 0)
		this.timeStarted = timeNowMsec;
	var timeElapsedMSec = timeNowMsec - this.timeStarted;
	
	var me = this;
	//console.log ("TaskDataAudit fase[" +this.fase +"]");
	switch (this.fase)
	{
		case 0:
			rhea.onEvent_readDataAudit = function(status, kbSoFar, fileID) 
										{ 
											//console.log("status[" +status +"], kbSoFar[" +kbSoFar +"], fileID[" +fileID +"]"); 
											switch (status)
											{
											case 0://in progress
												pleaseWait_freeText_setText ("DOWNLOADING EVA-DTS: " +kbSoFar +" Kb");
												break;
												
											case 1: //finished ok
												me.fase = 10;
												me.fileID = fileID;
												break;
											
											default:	//errore
												me.status = 200;
												break;
											}
										}
			this.fase = 1;
			pleaseWait_show();
			pleaseWait_freeText_setText ("REQUESTING EVA-DTS, please wait");
			pleaseWait_freeText_show();
			rhea.sendStartDownloadDataAudit();			
			break;
			
		case 1: //download eva-dts in corso
			break;
			
		case 10: //eva dts scaricato
			rhea.onEvent_readDataAudit = function(status, kbSoFar, fileID) {};
			me.fase = 20;
			pleaseWait_freeText_appendText ("<br>Gata. Procesare date, te rog așteaptă<br>"); //Done, processing data, please wait
			break;
			
		case 20: //inizio il download della versione "packed" dell'eva-dts che la GPU ha generato durante la fase precedente
			me.fase = 21;
			rhea.filetransfer_startDownload ("packaudit" +me.fileID, me, TaskDataAudit_load_onStart, TaskDataAudit_load_onProgress, TaskDataAudit_load_onEnd);
			break;
			
		case 21: //attende fine download file packed
			break;
			
			
			
		case 200: //errore downloading eva-dts
			rhea.onEvent_readDataAudit = function(status, kbSoFar, fileID) {};
			pleaseWait_freeText_appendText("Eroare în timpul descărcării EVA-DTS. Te rog încearcă din nou mai târziu"); //Error downloading EVA-DTS. Please try again later
			me.fase = 201;
			break;
			
		case 201: //mostra btn close e ne aspetta la pressione
			pleaseWait_btn1_setText("ÎNCHIDE");
			pleaseWait_btn1_show();
			me.fase = 202;
			break;			
		case 202: //attende pressione btn1
			break;

			
		case 210: //fine
			this.fase = 211;
			pageDataAudit_downloadEVA_onFinish();
			break;
			
		default:
			break;
	}
}


function TaskDataAudit_load_onStart(userValue)					{ }
function TaskDataAudit_load_onProgress()						{ }
function TaskDataAudit_load_onEnd (theTask, reasonRefused, obj)
{
	if (reasonRefused != 0)
	{
		pleaseWait_freeText_appendText ("Error, reason[" +reasonRefused +"]");
		theTask.fase = 201;
		return;
	}

	theTask.buffer = new Uint8Array(obj.fileSize);
	theTask.bufferSize = parseInt(obj.fileSize);
	for (var i=0; i<obj.fileSize; i++)
		theTask.buffer[i] = obj.fileBuffer[i];
	theTask.fase = 210;
}


/********************************************************
 * TaskDAResetTotals
 */
function TaskDAResetTotals()
{
	this.fase = 0;	
}
TaskDAResetTotals.prototype.onEvent_cpuStatus 	= function(statusID, statusStr, flag16)	{}
TaskDAResetTotals.prototype.onEvent_cpuMessage 	= function(msg, importanceLevel)		{ rheaSetDivHTMLByName("footer_C", msg); }
TaskDAResetTotals.prototype.onFreeBtn1Clicked	= function(ev)							{ pleaseWait_btn1_hide(); pleaseWait_btn2_hide(); if (this.fase==11) this.fase = 20;}
TaskDAResetTotals.prototype.onFreeBtn2Clicked	= function(ev)							{ pleaseWait_btn1_hide(); pleaseWait_btn2_hide(); this.fase = 90;}
TaskDAResetTotals.prototype.onFreeBtnTrickClicked= function(ev)							{}
TaskDAResetTotals.prototype.onTimer = function(timeNowMsec)
{
	var me = this;
	switch (this.fase)
	{
	case 0:
		this.fase = 1;
		pleaseWait_freeText_setText("WARNING: This procedure will RESET all EVADTS total counters.<br>It is highly recommended NOT to do this operation.<br><br>If you're sure you know what you're doing, click RESET ALL TOTAL COUNTERS button, otherwise click CANCEL");
		pleaseWait_freeText_show();
		
		pleaseWait_btn1_hide();
		pleaseWait_btn2_setText("CANCEL");
		pleaseWait_btn2_show();
		break;
		
	//aspetto qualche secondo prima di far vedere il bt "RESET ALL TOTAL"
	case 1: this.fase=2; break;
	case 2: this.fase=3; break;
	case 3: this.fase=4; break;
	case 4: this.fase=5; break;
	case 5: this.fase=10; break;
	
	case 10:
		this.fase = 11;
		pleaseWait_btn1_setText("RESET ALL TOTAL COUNTERS");
		pleaseWait_btn1_show();
		break;
		
	case 11: //attendo pressione di btn1 o 2
		break;
		
	case 20:
		//ho premuto btn1
		this.fase = 90;
		rhea.ajax ("EVArstTotals", "")
			.then( function(result) 
			{
				me.fase = 90;
			})
			.catch( function(result)
			{
				me.fase = 90;				
			});
		break;
	
	case 90:
		this.fase = 99;
		pageDataAudit_showSecretDaResetButtonWindow_finished();
		break;
	}
}


/********************************************************
 * TaskResetEVA
 */
function TaskResetEVA()																{ this.fase = 0;}
TaskResetEVA.prototype.onEvent_cpuStatus 	= function(statusID, statusStr, flag16)	{}
TaskResetEVA.prototype.onEvent_cpuMessage 	= function(msg, importanceLevel)		{ rheaSetDivHTMLByName("footer_C", msg); }
TaskResetEVA.prototype.onFreeBtn1Clicked	= function(ev)							{ pleaseWait_btn1_hide(); pleaseWait_btn2_hide(); this.fase = 10; }
TaskResetEVA.prototype.onFreeBtn2Clicked	= function(ev)							{ pleaseWait_btn1_hide(); pleaseWait_btn2_hide(); this.fase = 99; }
TaskResetEVA.prototype.onFreeBtnTrickClicked= function(ev)							{}
TaskResetEVA.prototype.onTimer 				= function(timeNowMsec)					
{
	//console.log ("TaskResetEVA::fase[" +this.fase +"]");
	switch (this.fase)
	{
	case 10: //do reset
		this.fase = 11;
		var me = this;		
		rhea.ajax ("EVArst", "")
			.then( function(result) 
			{
				me.fase = 90;
			})
			.catch( function(result)
			{
				me.fase = 99;				
			});
		break;
	
	case 11:
		break;
		
	case 90:
		rheaSetDivHTMLByName("pageDataAudit_lastDownload", "");
		rheaSetDivHTMLByName("pageDataAudit_o", "");
		rheaSetDivHTMLByName("pageDataAudit_t", "");
		rheaSetDivHTMLByName("pageDataAudit_p", "");
		this.fase = 99;
		break;

		
	case 99: //fine
		pleaseWait_hide();
		this.fase = 0;
		
		break;
		
	default:
		break;
		
	}
}


/********************************************************
 * TaskP15
 */
function TaskP15()																{ this.nextTimeSendP15 = 0;}
TaskP15.prototype.onEvent_cpuStatus 	= function(statusID, statusStr, flag16)	{}
TaskP15.prototype.onEvent_cpuMessage 	= function(msg, importanceLevel)		{ rheaSetDivHTMLByName("footer_C", msg); }
TaskP15.prototype.onFreeBtn1Clicked	= function(ev)								{}
TaskP15.prototype.onFreeBtn2Clicked	= function(ev)								{}
TaskP15.prototype.onFreeBtnTrickClicked= function(ev)							{}
TaskP15.prototype.onTimer 				= function(timeNowMsec)					
{
	if (timeNowMsec >= this.nextTimeSendP15)
	{
		this.nextTimeSendP15 = timeNowMsec + 5000;
		var buffer = new Uint8Array(1);
		buffer[0] = 66;
		rhea.sendGPUCommand ("E", buffer, 0, 0);		
		//console.log ("p15");
	}
}




/**********************************************************
 * TaskEspressoCalib
 */
function TaskEspressoCalib()
{
	this.what = 0;
	this.fase = 0;
	this.cpuStatus = 0;
	this.selNum = 0;

	this.setMacina(1);
	this.enterQueryMacinePos();
}

TaskEspressoCalib.prototype.setMacina = function (macina1o2)
{
	this.macina1o2 = macina1o2;
	this.firstTimeMacina = 2;
	var w = ui.getWindowByID("pageExpCalib");
	w.getChildByID("pageExpCalib_vgBtnSet").hide();	
}

TaskEspressoCalib.prototype.enterQueryMacinePos = function()
{	
	this.what = 0;
	this.fase = 0;
}

TaskEspressoCalib.prototype.enterSetMacinaPos = function(targetValue)
{	
	this.what = 1;
	this.fase = 0;
	pleaseWait_show();
	rhea.sendStartPosizionamentoMacina(this.macina1o2, targetValue);
}

TaskEspressoCalib.prototype.runSelection = function(selNum)
{	
	this.what = 2;
	this.fase = 0;
	this.selNum = selNum;
	pleaseWait_show();
}

TaskEspressoCalib.prototype.messageBox = function (msg)
{
	this.what = 3;
	this.fase = 0;
	pleaseWait_show();
	pleaseWait_rotella_hide();
	pleaseWait_btn1_setText("OK");
	pleaseWait_btn1_show();
	pleaseWait_freeText_show();
	pleaseWait_freeText_setText(msg);
	
}


TaskEspressoCalib.prototype.onEvent_cpuStatus  = function(statusID, statusStr, flag16)	{ this.cpuStatus = statusID; pleaseWait_header_setTextL(statusStr); }
TaskEspressoCalib.prototype.onEvent_cpuMessage = function(msg, importanceLevel)		{ rheaSetDivHTMLByName("footer_C", msg); pleaseWait_header_setTextR(msg); }
TaskEspressoCalib.prototype.onFreeBtn1Clicked	 = function(ev)
{
	switch (this.what)
	{
	case 1:
		//siamo in regolazione apertura vgrind
		if (this.fase > 0)
		{
			pleaseWait_btn1_hide();
			
			//Attivo la macina
			rhea.ajax ("runMotor", { "m":10+this.macina, "d":50, "n":1, "p":0}).then( function(result)
			{
				setTimeout ( function() {pleaseWait_btn1_show();}, 5000);
			})
			.catch( function(result)
			{
				pleaseWait_btn1_show();
			});								
		}
		break;
		
	case 3: //message box
		pleaseWait_hide();
		this.what = 0;
		break;
	}
}
TaskEspressoCalib.prototype.onFreeBtn2Clicked	 = function(ev)						{}
TaskEspressoCalib.prototype.onFreeBtnTrickClicked= function(ev)						{}


TaskEspressoCalib.prototype.onTimer = function (timeNowMsec)
{
	if (this.what == 0)
		this.priv_handleRichiestaPosizioneMacina();
	else if (this.what == 1)
		this.priv_handleRegolazionePosizioneMacina();
	else if (this.what == 2)
		this.priv_handleRunSelection(timeNowMsec);
}

TaskEspressoCalib.prototype.priv_handleRunSelection = function(timeNowMsec)
{
	if (this.fase == 0)
	{
		this.fase = 1;
		this.timeStartedMSec = timeNowMsec;
		
		rhea.ajax ("testSelection", {"s":this.selNum, "d":0} ).then( function(result)
		{
			if (result != "OK")
			{
				this.fase = 2;
				pageExpCalib_runSelection_onFinish();
			}
		})
		.catch( function(result)
		{
			this.fase = 2;
			pageExpCalib_runSelection_onFinish();
		});			
	}
	else if (this.fase == 1)
	{
		//aspetto almeno un paio di secondi
		if ((timeNowMsec - this.timeStartedMSec) < 2000)
			return;
		//monitoro lo stato di cpu per capire quando esce da 3 (prep bevanda)
		if (this.cpuStatus != 3 && this.cpuStatus != 101 && this.cpuStatus != 105)
		{
			this.fase = 2;
			pageExpCalib_runSelection_onFinish();
		}
	}
	
}

TaskEspressoCalib.prototype.priv_handleRichiestaPosizioneMacina = function()
{
	if (da3.isGruppoMicro())
		return;
	var me = this;
	if (this.fase == 0)
	{
		//chiede la posizione della macina
		this.fase = 1;
		if (da3.isGruppoVariflex())
		{
			rhea.ajax ("getPosMacina", {"m":this.macina1o2}).then( function(result)
			{
				var obj = JSON.parse(result);
				rheaSetDivHTMLByName("pageExpCalib_vgCurPos", obj.v);
				if (me.firstTimeMacina>0)
				{
					me.firstTimeMacina--;
					if (me.firstTimeMacina==0)		
					{
						var w = ui.getWindowByID("pageExpCalib");
						w.getChildByID("pageExpCalib_vg_target").setValue(obj.v)
						w.getChildByID("pageExpCalib_vgBtnSet").show();
					}
				}
				me.fase = 0;
			})
			.catch( function(result)
			{
				me.fase = 0;
			});			
		}
		return;
	}
	else
	{
		//aspetto una risposta alla query precedente
	}

}

TaskEspressoCalib.prototype.priv_handleRegolazionePosizioneMacina = function()
{
	switch (this.fase)
	{
		case 0: 
			this.fase=1; 
			
			pleaseWait_freeText_setText("Te rog așteaptă ca varigrind să își ajusteze poziția"); //Please wait while the varigrind is adjusting its position
			pleaseWait_freeText_show();
			break;
		case 1: this.fase=2; break;
		case 2: 
			this.priv_queryMacina();
			this.fase=3; 
			break;
		
		case 3: 
			//a questo punto CPU dovrebbe essere in stato 102 e dovrebbe rimanerci fino a fine operazione
			if (this.cpuStatus != 102 && this.cpuStatus != 101)
			{
				this.enterQueryMacinePos();
				pleaseWait_hide();
				return;
			}
			
			this.fase = 2;
			break;
	}
}

TaskEspressoCalib.prototype.priv_queryMacina = function()
{
	if (!da3.isGruppoVariflex())
		return;
	rhea.ajax ("getPosMacina", {"m":this.macina1o2}).then( function(result)
	{
		var obj = JSON.parse(result);
		rheaSetDivHTMLByName("pageExpCalib_vgCurPos", obj.v);
		pleaseWait_freeText_setText("Te rog așteaptă ca varigrind să își ajusteze poziția" +"  [" +obj.v +"]"); //Please wait while the varigrind is adjusting its position [current_value]
	})
	.catch( function(result)
	{
	});			
}


/**********************************************************
 * TaskGrinderClean
 */
function TaskGrinderClean ()
{
	this.fase = 0;
	this.isBuzzing = 0;
	this.buzzer_nextTimeAskMSec = 0;
}

TaskGrinderClean.prototype.onEvent_cpuStatus  	= function(statusID, statusStr, flag16)		{ this.cpuStatus = statusID; pleaseWait_header_setTextL ("CURĂȚARE RÂȘNIȚĂ"); }
TaskGrinderClean.prototype.onEvent_cpuMessage 	= function(msg, importanceLevel)			{ rheaSetDivHTMLByName("footer_C", msg); pleaseWait_header_setTextR(msg); }
TaskGrinderClean.prototype.finished 			= function () 								{ pleaseWait_hide(); this.fase=0; pageGrinderCleaning_goBack(); }

TaskGrinderClean.prototype.onTimer 				= function(timeNowMsec)						
{ 
	switch (this.fase)
	{
		case 41: this.step41(timeNowMsec); break;
		case 203: this.runGrinderCycle_3(timeNowMsec); break;
	}
}

TaskGrinderClean.prototype.pollBuzzerStatus = function()
{
	var me = this;
	rhea.ajax ("getBuzzerStatus", "").then( function(result)
	{
		if (result=="IDLE")
		{
			me.isBuzzing = 0;
			if (me.btn1ShouldBeVisibileAfterBuzzEnd)
				pleaseWait_btn1_show();
			if (me.btn2ShouldBeVisibileAfterBuzzEnd)
				pleaseWait_btn2_show();
//console.log ("BUZZ END");
		}
		else
			setTimeout (me.pollBuzzerStatus(), 100);
		
	})
	.catch( function(result)
	{
		me.isBuzzing = 0;
		if (me.btn1ShouldBeVisibileAfterBuzzEnd)
			pleaseWait_btn1_show();
		if (me.btn2ShouldBeVisibileAfterBuzzEnd)
			pleaseWait_btn2_show();
//console.log ("ERR but BUZZ END");			
	});
}

TaskGrinderClean.prototype.beep = function (numRepeat)
{
	this.isBuzzing = 1;
	var buzzer_dsec = da3.read8(7064);
	rhea.activateCPUBuzzer (numRepeat, buzzer_dsec, 5);
//console.log ("BEEP for "+buzzer_dsec +" dsec x " +numRepeat);
	this.pollBuzzerStatus();
}


TaskGrinderClean.prototype.onFreeBtn1Clicked	= function(ev)						
{
	switch (this.fase)
	{
		default: return;
		case 1:	 this.step2();	break //btn continue
		case 2:	 this.step3();	break //btn continue
		case 3:	 this.step4();	break //btn continue
		case 5:	 this.step6();	break //btn continue
		case 6:	 this.step7();	break //btn continue
		case 20: this.step21();	break //btn NO
		case 21: this.step23();	break //btn continue
		case 23: this.step24();	break //btn continue
		case 25: this.step26();	break //btn NO
		case 26: this.step27();	break //btn continue
		case 28: this.step99();	break //btn NO
	}
}

TaskGrinderClean.prototype.onFreeBtn2Clicked = function()
{
	switch (this.fase)
	{
		default: return;
		case 1:	 this.finished();	break //btn abort
		case 20: this.step6();	break //btn YES
		case 25: this.step24();	break //btn YES
		case 28: this.step40();	break //btn YEs
	}
}

TaskGrinderClean.prototype.runGrinderCycle = function (numCicli, tempoGrinderONSec, tempoGrinderOFFSec, fnToCallOnFinish)
{
	//numCicli=2; tempoGrinderONSec=2; tempoGrinderOFFSec=2;
	
	this.fase = 200;
	this.numCicli = numCicli;
	this.tempoGrinderONSec = tempoGrinderONSec;
	this.tempoGrinderOFFSec = tempoGrinderOFFSec;
	this.fnToCallOnFinish = fnToCallOnFinish;

	this.curCiclo = 1;
	this.priv_show ("", "", "");	
	pleaseWait_rotella_show();
	this.runGrinderCycle_1();
}

TaskGrinderClean.prototype.runGrinderCycle_1 = function()
{
	//this.beep(1);
	this.fase = 201;

	//var msg = "Running grinder cycle " +this.curCiclo +" of " +this.numCicli;
	//msg += "<br><br><b>Catch the product</b>";
	var msg = "Ciclu râșniță în curs {0} din {1}".translateLang(this.curCiclo,this.numCicli);
	msg += "<br><br><b>Preia produsul</b>";
	this.priv_show (msg, "", "");	

	var me = this;
	rhea.ajax ("runMotor", { "m":10+this.grinder1o2, "d":this.tempoGrinderONSec*10, "n":1, "p":0}).then( function(result)
	{
		setTimeout ( function() { me.runGrinderCycle_2(); }, me.tempoGrinderONSec*1000 + 100);
	})
	.catch( function(result)
	{
		setTimeout ( function() { me.runGrinderCycle_1(); }, 500);
	});	
}

TaskGrinderClean.prototype.runGrinderCycle_2 = function()
{
	//se ho finito di fare le macinate, goto fine, altrimenti goto runGrinderCycle_1
	this.fase = 202;
	this.curCiclo++;
	if (this.curCiclo > this.numCicli)
		setTimeout ( this.fnToCallOnFinish(), 10);
	else
	{
		this.waitUntilMSec = 0;
		this.fase = 203; //la runGrinderCycle_3() viene chiamato dalla onTimer 1 volta al sec
	}
}

TaskGrinderClean.prototype.runGrinderCycle_3 = function(timeNowMsec)
{
	this.fase = 203;
	//devo aspettare un tot di secondi prima di riepetere la macinata dello runGrinderCycle_1 (vedi onTimer)
	if (this.waitUntilMSec == 0)
		this.waitUntilMSec = timeNowMsec + 1000 * this.tempoGrinderOFFSec;
	
	var timeLeftMSec = this.waitUntilMSec - timeNowMsec;
	var timeLeftSec= Math.floor(timeLeftMSec / 1000);
	
	//var msg = "Running grinder cycle " +(this.curCiclo-1) +" of " +this.numCicli;
	//msg += "<br>Waiting " +timeLeftSec +" sec.";
	var msg = "Ciclu râșniță în curs {0} din {1}".translateLang(this.curCiclo-1,this.numCicli);
	msg += "<br>" +"Așteaptă {0} sec.".translateLang(timeLeftSec);
	this.priv_show (msg, "", "");	
	if (timeLeftMSec <= 0)
		this.runGrinderCycle_1();
}

TaskGrinderClean.prototype.priv_show = function (text, text_btn1, text_btn2)
{
//console.log ("fase="+this.fase);	
	this.btn1ShouldBeVisibileAfterBuzzEnd = 0;
	this.btn2ShouldBeVisibileAfterBuzzEnd = 0;

	if (text_btn1 == "")
		pleaseWait_btn1_hide();
	else
	{
		pleaseWait_btn1_setText(text_btn1);
		pleaseWait_btn1_show();
		this.btn1ShouldBeVisibileAfterBuzzEnd = 1;
	}
	
	if (text_btn2 == "")
		pleaseWait_btn2_hide();
	else
	{
		pleaseWait_btn2_setText(text_btn2);
		pleaseWait_btn2_show();
		this.btn2ShouldBeVisibileAfterBuzzEnd = 1;
	}

	if (text == "")
		pleaseWait_freeText_hide();
	else
	{
		pleaseWait_freeText_setText ("<b>CURĂȚARE RÂȘNIȚĂ</b><br><br>" + text);
		pleaseWait_freeText_show();
		
	}

	if (this.isBuzzing && (text_btn1!="" || text_btn2!=""))
	{
		pleaseWait_btn1_hide();
		pleaseWait_btn2_hide();
	}
	
}


TaskGrinderClean.prototype.step1 = function()
{
	this.beep(3);
	this.fase = 1;
	pleaseWait_show();
	pleaseWait_rotella_hide();

	this.priv_show ("&nbsp;", "CONTINUĂ", "ANULEAZĂ");
	rheaShowElem (rheaGetElemByID("pagePleaseWait_grinderCleaning")); //mostra i btn per la scelta di grinder 1 o 2
}

TaskGrinderClean.prototype.step2 = function()
{
	this.beep(3);
	this.fase = 2;
	
	this.grinder1o2 = parseInt(uiStandAloneOptionGrinderCleaning1or2.getSelectedOptionValue());
	rheaHideElem(rheaGetElemByID("pagePleaseWait_grinderCleaning"));

	//Close bean hopper shutter to avoid loss of beans.<br>Press CONTINUE when done.
	var msg = "Închide capacul recipientului pentru cafea boabe pentru a evita pierderea boabelor.";
	msg += "<br>Apasă CONTINUARE la final.";
	this.priv_show (msg, "CONTINUĂ", "");
}

TaskGrinderClean.prototype.step3 = function()
{
	this.beep(3);
	this.fase = 3;
	//Remove brewer and bean hopper.<br>Press CONTINUE when done.
	var msg = "Scoate grupul cafea și recpientul de cafea boabe.";
	msg += "<br>Apasă CONTINUARE la final.";
	this.priv_show (msg, "CONTINUĂ", "");
}

TaskGrinderClean.prototype.step4 = function()
{
	//bisogna accertarsi che il gruppo sia scollegato
	this.fase = 4;
	this.priv_show ("", "", "");
	pleaseWait_rotella_show();

	var me = this;
	rhea.ajax ("getGroupState", "").then( function(result)
	{
		pleaseWait_rotella_hide();
		if (result=="0")
			me.step5();
		else
			me.step3();
	})
	.catch( function(result)
	{
		me.step3();
	});			
}

TaskGrinderClean.prototype.step5 = function()
{
	this.beep(3);
	this.fase = 5;
	pleaseWait_rotella_hide();
	//Install grinder cleaning device.<br>Press CONTINUE when done.
	var msg = "Instalează dispozitivul de curățare râșniță.";
	msg += "<br>Apasă CONTINUARE la final.";	
	this.priv_show (msg, "CONTINUĂ", "");	
}

TaskGrinderClean.prototype.step6 = function()
{
	this.beep(3);
	this.fase = 6;
	
	//var msg = "Refill cleaning device.<br>When done, press CONTINUE.";
	//msg += "<br><br><b>WARNING:</b> as soon as you press CONTINUE, the grinder will start running";
	var msg = "Umple dispozitivul de curățare.";
	msg += "<br>Apasă CONTINUARE la final.";
	msg += "<br><br><b>ATENȚIE:</b> imediat ce apeși CONTINUARE, râșnița va începe să funcționeze";
	this.priv_show (msg, "CONTINUĂ", "");	
}

TaskGrinderClean.prototype.step7 = function()
{
	this.fase = 7;
	this.runGrinderCycle (5, 5, 10, this.step20);
}

TaskGrinderClean.prototype.step20 = function()
{
	this.beep(3);
	this.fase = 20;
	pleaseWait_rotella_hide();
	//Do you want to repeat the grinding cycles?
	this.priv_show ("Vrei să repeți ciclurile de măcinare?", "NU", "DA");
}

TaskGrinderClean.prototype.step21 = function()
{
	this.beep(3);
	this.fase = 21;
	
	//"Put back  bean hopper. Press CONTINUE when done."
	var msg = "Pune înapoi recipientul pentru boabe.";
	msg += "<br>Apasă CONTINUARE la final.";
	this.priv_show (msg, "CONTINUĂ", "");
}

TaskGrinderClean.prototype.step23 = function()
{
	this.beep(3);
	this.fase = 23;
	//Open hopper shutter. When done, press CONTINUE.<br><br><b>WARNING:</b> as soon as you press CONTINUE, the grinder will start running.
	var msg = "Deschide capacul recipientului.";
	msg += "<br>Apasă CONTINUARE la final.";
	msg += "<br><br><b>ATENȚIE:</b> imediat ce apeși CONTINUARE, râșnița va începe să funcționeze";
	this.priv_show (msg, "CONTINUĂ", "");	
}

TaskGrinderClean.prototype.step24 = function()
{
	this.fase = 24;
	this.runGrinderCycle (5, 5, 10, this.step25);	
}

TaskGrinderClean.prototype.step25 = function()
{
	this.beep(3);
	this.fase = 25;
	pleaseWait_rotella_hide();
	
	//Do you want to repeat the grinding cycles?
	this.priv_show ("Vrei să repeți ciclurile de măcinare?", "NU", "DA");
}


TaskGrinderClean.prototype.step26 = function()
{
	this.beep(3);
	this.fase = 26;
	//Put brewer back into position, press CONTINUE when done.
	var msg = "Pune grupul cafea înapoi pe poziție.";
	msg += "<br>Apasă CONTINUARE la final.";
	this.priv_show (msg, "CONTINUĂ", "");
}


TaskGrinderClean.prototype.step27 = function()
{
	//bisogna accertarsi che il gruppo sia collegato
	this.fase = 27;
	this.priv_show ("", "", "");
	pleaseWait_rotella_show();

	var me = this;
	rhea.ajax ("getGroupState", "").then( function(result)
	{
		pleaseWait_rotella_hide();
		if (result=="1")
			me.step28();
		else
			me.step26();
	})
	.catch( function(result)
	{
		me.step26();
	});			
}

TaskGrinderClean.prototype.step28 = function()
{
	this.beep(3);
	this.fase = 28;
	pleaseWait_rotella_hide();
	
	//Do you want do dispense a coffe?
	this.priv_show ("Vrei să faci o cafea?", "NU", "DA");
}

TaskGrinderClean.prototype.step40 = function()
{
	//bisogna far partire un coffe
	this.fase = 40;
	this.priv_show ("", "", "");
	pleaseWait_rotella_show();

	this.hoVistoCPUInStatoPREP_BEVANDA = 0;
	this.fase = 41;
	rhea.ajax ("runCaffeCortesia", {"m":this.grinder1o2}).then( function(result)
	{
	})
	.catch( function(result)
	{
	});		
	

}

TaskGrinderClean.prototype.step41 = function(timeNowMsec)	
{
	//questa viene chiamata periodicamente dalla onTime
	//Rimango qui fino a che la CPU non passa dallo stato PREPARAZIONE_BEVANDA(3) allo stato DISPONIBILE(2)
	if (this.hoVistoCPUInStatoPREP_BEVANDA == 0)
	{
		if (this.cpuStatus == 3)
			this.hoVistoCPUInStatoPREP_BEVANDA = 1;
	}
	else
	{
		if (this.cpuStatus != 3)
			this.step99();
	}		
}
	
TaskGrinderClean.prototype.step99 = function() //fine
{
	//Segnalo a CPU che ho finito la procedura
	rhea.ajax ("notifyEndGrinClean", {"m":this.grinder1o2} ).then( function(result)
	{
	})
	.catch( function(result)
	{
	});		

	this.beep(5);
	this.finished();
}

/*************************************************************
 * TaskCalibMilkPump		//@TP - #1885
 *************************************************************/
function TaskCalibMilkPump()
{
	this.timeStarted = 0;
	this.what = 1;  //0==nulla, 1=procedura di tuning pompa latte
	this.fase = 0;
	this.value = 0;
	this.cpuStatus = 0;
	this.savFase = 0;
}

TaskCalibMilkPump.prototype.onEvent_cpuStatus		= function(statusID, statusStr, flag16)	{ this.cpuStatus = statusID; pleaseWait_header_setTextL (statusStr); }
TaskCalibMilkPump.prototype.onEvent_cpuMessage		= function(msg, importanceLevel)		{ rheaSetDivHTMLByName("footer_C", msg); pleaseWait_header_setTextR(msg);}
TaskCalibMilkPump.prototype.onFreeBtnTrickClicked	= function(ev)							{}
TaskCalibMilkPump.prototype.onFreeBtn1Clicked		= function(ev)
{ 
	if (this.fase == 10)
	{
		pleaseWait_btn1_hide(); 
		pleaseWait_btn2_hide();
		this.fase = 12;				//aggiunta fase caricamento tubo 
	}	
	
	if (this.fase == 14)
	{
		pleaseWait_btn1_hide(); 
		pleaseWait_btn2_hide();
		this.fase = 15;				//aggiunta fase caricamento tubo 
	}	
	
	if (this.fase == 30)
	{
		this.fase = 40;
	}	
}

TaskCalibMilkPump.prototype.onFreeBtn2Clicked	= function(ev)
{ 
		pleaseWait_btn1_hide(); 
		pleaseWait_btn2_hide(); 
		this.fase = 200;  	
}

TaskCalibMilkPump.prototype.startCalibMilkPump = function()
{
	this.timeStarted = 0;
	this.what = 1;  //0==nulla, 1=procedura di tuning pompa latte
	this.fase = 0;
	this.value = 0;
	this.cpuStatus = 0;
	this.savFase = 0;
	pleaseWait_calibMilkPump_hide();
}

TaskCalibMilkPump.prototype.onTimer = function (timeNowMsec)
{
	if (this.timeStarted == 0)
		this.timeStarted = timeNowMsec;
	
	var timeElapsedMSec = timeNowMsec - this.timeStarted;

	this.priv_handleCalibMilkPump(timeElapsedMSec);	
}

TaskCalibMilkPump.prototype.priv_handleCalibMilkPump = function (timeElapsedMSec)
{
	var TIME_ATTIVAZIONE_dSEC = 40;
	var TIME_ATTIVAZIONE_FOR_TUN_dSEC = 200;
	
	var me = this;

	if (this.what == 0) return;    //ho finito la calibrazione ed entrerò in questa funzione solo se premo nuovamente il tasto 'calibrate'
	
	switch (this.fase)
	{
	case 0:
		me.fase = 10;
		pleaseWait_show();
		pleaseWait_calibMilkPump_show();
		pleaseWait_calibMilkPump_setText("Umple cu apă vasul de lapte, pune o carafă în stația pentru pahare și apasă CONTINUARE");
		pleaseWait_btn1_setText("CONTINUĂ");
		pleaseWait_btn1_show();
		var btnText = "IEȘI";
		pleaseWait_btn2_setText (btnText);
		pleaseWait_btn2_show();				
		break;
		
	case 10:			//Attendo che venga premuto 'CONTINUE' o 'ABORT'
		break;
	
	case 12:
		me.fase = 13;
		pleaseWait_show();
		pleaseWait_calibMilkPump_show();
	    pleaseWait_calibMilkPump_setText("Așteaptă să fie furnizată apa"); 
		rhea.ajax ("runMilkPump", {"d":TIME_ATTIVAZIONE_dSEC, "n":2, "p":10}).then( function(result)
		{
			setTimeout ( function() { me.fase=13; }, TIME_ATTIVAZIONE_dSEC*2*100);  // - 1000);
		})
		.catch( function(result)
		{
			me.fase = 200;
		});		
		break;	
		
	case 13:
		me.fase = 14;
		pleaseWait_show();
		pleaseWait_calibMilkPump_show();
		pleaseWait_calibMilkPump_setText("Golește carafa din stația pentru pahare și apasă CONTINUARE");
		pleaseWait_btn1_setText("CONTINUĂ");
		pleaseWait_btn1_show();
		break;	
	
	case 14:
		break;	
		
	case 15:
		me.fase = 20;
		pleaseWait_show();
		pleaseWait_calibMilkPump_show();
	    pleaseWait_calibMilkPump_setText("Așteaptă să fie furnizată apa"); 
		rhea.ajax ("runMilkPump", {"d":TIME_ATTIVAZIONE_FOR_TUN_dSEC, "n":2, "p":10}).then( function(result)
		{
			setTimeout ( function() { me.fase=20; }, TIME_ATTIVAZIONE_FOR_TUN_dSEC*2*100); // - 1000);
		})
		.catch( function(result)
		{
			me.fase = 200;
		});		
		break;	
		
	case 20:
		me.fase = 30;
		
		//mostro la riga che chiede se si vuole erogare nuovamente l'acqua da misurare
		pleaseWait_calibMilkPump_dispenseWater_show();	
		pleaseWait_calibMilkPump_setText("La final, introdu cantitatea în ml de apă furnizată, <br> max 340ml <br>apoi apasă CONTINUARE<br><br>"); //Please enter the quantity, then press CONTINUE
		pleaseWait_calibMilkPump_num_setValue(0);
		pleaseWait_calibMilkPump_num_show();
		pleaseWait_btn1_setText("CONTINUĂ");
		pleaseWait_btn1_show();
		break;
		
	case 30: 		//Attendo che venga premuto 'CONTINUE' dopo aver inserito i ml erogati
		break;
		
	case 40:
		pleaseWait_calibMilkPump_dispenseWater_hide();
		me.value = pleaseWait_calibMilkPump_num_getValue();
		if ((parseFloat(me.value) < 255) || (parseFloat(me.value) > 474))
		{
			pleaseWait_calibMilkPump_setText("Invalid value");
			me.fase = 20;
			break;
		}
	
		pleaseWait_calibMilkPump_setText("Storing value ...");
		me.value = pleaseWait_calibMilkPump_num_getValue();
		pleaseWait_calibMilkPump_num_hide();
		
        var val_rif_pump_milk = parseInt((33200 / (parseInt(me.value))));
		if ((val_rif_pump_milk <70) || (val_rif_pump_milk >130)) val_rif_pump_milk=100;
		var com_indirizzo=8366;	
		da3.write16(com_indirizzo, parseInt(val_rif_pump_milk));
		me.fase = 200;
		break;
		
	case 200:
	    me.fase = 0;
		me.what = 0;
		pleaseWait_btn1_hide();
		pleaseWait_calibMilkPump_hide();
		pageMilker_tuningPumpOnFinish();
		break;
	}
}

//------- Cleaning del modulo sciroppi -------------
TaskCleaning.prototype.priv_handleSyrupCleaning = function (timeElapsedMSec)		//#4660
{
	//termino quando lo stato della CPU diventa != da SYRUP_CLEANING
	if (timeElapsedMSec > 3000 && this.cpuStatus != 31) //31==syrup Cleaning
	{
		pageCleaning_onFinished();
		return;
	}
	
	//ogni tot mando una richiesta per conoscere lo stato attuale del lavaggio
	if (timeElapsedMSec < this.nextTimeSanWashStatusCheckMSec)
		return;
	this.nextTimeSanWashStatusCheckMSec += 2000;	

	//periodicamente richiedo lo stato del lavaggio
	var me = this;
	rhea.ajax ("sanWashStatus", "")
		.then( function(result) 
		{
			var obj = JSON.parse(result);
			//console.log ("DESCALING MILKER response: fase[" +obj.fase +"] b1[" +obj.btn1 +"] b2[" +obj.btn2 +"]");
			me.fase = parseInt(obj.fase);
			me.btn1 = parseInt(obj.btn1);
			me.btn2 = parseInt(obj.btn2);
			var msg = "";
			switch (me.fase)
			{
				default: 	msg = " "; break;
				case 1:		msg = "Place an one-liter tank with the correct quantity of detergent under the cup station, Apasă CONTINUARE la final."; break;			//SYRUP_CLEANING_PREPARE_TANK_UNDER_NOZZLES,
				case 2:		msg = "Hot water is flowing from the the nozzle in the tank with cleaning detergent. te rog așteaptă…..."; break;
				case 3:		msg = "Remove the tubes from the syrup bottles and place them in the cleaning solution tank, Apasă CONTINUARE la final."; break;
				case 4:		msg = "step 1 - Cleaning solution is flowing from the syrup tube nozzles. te rog așteaptă…..."; break;
				case 5:		msg = "step 1 - Please wait for the cleaning solution action…, te rog așteaptă…..."; break;
				case 6:		msg = "step 2 - Cleaning solution is flowing from the syrup tube nozzles. te rog așteaptă…..."; break;
				case 7:		msg = "step 2 - Please wait for the cleaning solution action…. te rog așteaptă…..."; break;
				case 8:		msg = "Place a clean empty tank under the cup station, Apasă CONTINUARE la final."; break;
				case 9:		msg = "Hot water is flowing from the the nozzle in the tank. te rog așteaptă…..."; break;
				case 10:	msg = "Place the syrup tubes in the cleaning water tank. Apasă CONTINUARE la final."; break;
				case 11:	msg = "Hot water  is flowing from the syrup tube nozzles."; break;
				case 12:	msg = "Check the syrup tubes. Do you want to repeat the rinsing ?  Press REPEAT to rinse again the tubes.. Apasă CONTINUARE la final."; break;
				case 13:	msg = "Put the syrup tubes into the syrup bottles."; break;
			}
			
			//msg += "<br><br>DEBUG-FASE[" +me.fase +"] BTN1{" +me.btn1 +"} BTN2[" +me.btn2 +"]";
			
			pleaseWait_freeText_setText(msg);
			pleaseWait_freeText_show();
			
			if ( (me.fase != me.prevFase) || ( me.fase == me.prevFase && (me.btn1 != me.prevFaseBtn1 || me.btn2 != me.prevFaseBtn2) ))
			{
				me.prevFase = me.fase;
				me.prevFaseBtn1 = me.btn1;
				me.prevFaseBtn2 = me.btn2;

				
				if (me.btn1 == 0)
					pleaseWait_btn1_hide();
				else
				{
					var btnText = "BUTON " +me.btn1;
					switch (me.fase)
					{
						case 1:
						case 3:
						case 8:
						case 10:
						case 12:
							btnText = "CONTINUĂ"; 
							break;
						
						case 13:	
						case 14:
							btnText = "ÎNCHIDE"; 
							break;
					}
					pleaseWait_btn1_setText (btnText);
					pleaseWait_btn1_show();	
				}
				
				if (me.btn2 == 0)
					pleaseWait_btn2_hide();
				else
				{
					var btnText = "BUTON " +me.btn2;
					switch (me.fase)
					{
						case 12:
							btnText = "REPETĂ";
							break;
					}
					pleaseWait_btn2_setText (btnText);
					pleaseWait_btn2_show();	
				}				
			}				
		})
		.catch( function(result)
		{
			//console.log ("SYRUP: error[" +result +"]");
			//pleaseWait_btn1_hide();
			//pleaseWait_btn2_hide();
		});	
		
}



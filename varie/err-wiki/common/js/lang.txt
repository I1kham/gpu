var objLang = null;

function menuLang_setup (arrayOfAvailLang)	{ objLang = new ObjLang(arrayOfAvailLang); }
function menuLang_show(bShow)				{ objLang.show(bShow); }
function menuLang_onLangSelected(i)			{ objLang.onLangSelected(i); }

function ObjLang()
{
	this.langNames = [];

	this.langNames["GB"] = "ENGLISH";
	this.langNames["IT"] = "ITALIANO";
	this.langNames["FR"] = "FRANÇAIS";
	this.langNames["DE"] = "DEUTSCH";
	this.langNames["ES"] = "ESPAÑOL";
	this.langNames["PT"] = "PORTUGUÉS";
	this.langNames["TR"] = "TÜRKÇE";
	this.langNames["HU"] = "MAGYAR";	//ungherese -->
	this.langNames["CN"] = "中文";
	this.langNames["RU"] = "РУССКИЙ";
	this.langNames["RO"] = "ROMÂNĂ";
	this.langNames["NL"] = "NEDERLANDS";
	this.langNames["PL"] = "POLSKI";
	this.langNames["JP"] = "日本";
	this.langNames["CZ"] = "ČEŠTINA";	//czech -->
	this.langNames["KO"] = "한글어";		//koreano -->
	this.langNames["BG"] = "БЪЛГАРСКИ";	//bulgaro -->
	this.langNames["UA"] = "УКРАЇНСЬКА";//ucraino -->
	this.langNames["HR"] = "HRVATSKI";	//croato -->	
	
	

	this.listOfAvailLang = lang_listOfAvailLang;
	
	this.priv_buildHTML();
}


ObjLang.prototype.priv_buildHTML = function ()
{
	var n = this.listOfAvailLang.length;
	var html = "";
	
	
	var N_MAX_VOCI_PER_COLONNA = 7;
	var nVociInCurColonna = 0;
	for (var i=0; i<n; i++)
	{
		if (nVociInCurColonna == 0)
		{
			html += "<div style='float:left; width:33%'><ul>";
		}
		
		var flag = this.listOfAvailLang[i];
		var label = this.langNames[flag];
		html += "<li onmousedown='menuLang_onLangSelected(" +i +")'><img src='../../prog/img/flags/" +flag +".png'> " +label +"</li>";
		nVociInCurColonna++;
	
		if (nVociInCurColonna >= N_MAX_VOCI_PER_COLONNA)
		{
			html += "</ul></div>";
			nVociInCurColonna = 0;
		}
	}
		
	if (nVociInCurColonna > 0)
		html += "</ul></div>";
	
	html += "<div style='position:absolute; top:5px; right:5px'><img src='../../prog/img/close32x32.png' style='width:48px' draggable='false' onmousedown='menuLang_show(0)'></div>";
	
	rheaSetElemHTML (rheaGetElemByID("divMenuLang"), html);
}

ObjLang.prototype.show = function(bShow)
{
	var e = rheaGetElemByID("divMenuLang");
	if (bShow)
		rheaShowElem(e);
	else
		rheaHideElem(e);
}

ObjLang.prototype.onLangSelected = function (i)
{
	var iso = this.listOfAvailLang[i];
	window.location = "index_" +iso +".html";
}

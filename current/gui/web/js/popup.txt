var POPUP = {
    MILK: 0,
    WENGLOR: 1,
    FIRST_COFFEE: 2
};

function getPopup(type) {
    var content = '';

    switch (type) {
        case POPUP.MILK:
            content = ''
                + '<div class="milkBgImage"></div>'
                + '<div class="milkTextualContent">'
                +     '<div id="modalTitle" class="modalTitle" style="text-align: center; color: #db783e;">MILK Tank empty</div>'
                +     '<div id="modalDescription" class="modalDescription" style="text-align: center;margin-top: 20px; margin-bottom: 55px;">'
                +        'please press the button<br>"Confirm" or "Cancel" below:'
                +     '</div>'
                + 	  '<div class="buttonContainer" style="position: absolute;bottom: 0;left: 0;right: 0;">'
                +        '<div id="btnMilkConfirm" class="btnAction" onclick="milkPopup_confirm()">'
                +	     '<p id="btnMilkConfirmLabel">Confirm: if you have refilled the milk</p>'
                +	  '</div>'
                +	  '<div id="btnMilkCancel" class="btnAction" onclick="milkPopup_cancel()">'
                +	     '<p id="btnMilkCancelLabel">Cancel: if you will do it late</p>'
                +	  '</div>'	
				+ '</div>';
            
                break;


        case POPUP.WENGLOR:
            content = ''
            +     '<div id="modalCupTitle" class="modalCupTitle" style="text-align: center;margin-top: 80px;font-size: 25px;"></div>'
            +	  '<div id="btnCupConfirm" class="btnCupConfirm" onclick="cupConfirmPopup_cancel()" style="display: table; width: 240px; height: 37px; border: 1px solid; border-radius: 25px; color: #838383; font-weight: bold; position: absolute; bottom: 15px; left: 0; right: 0; margin: auto; text-align: center; cursor: pointer; margin-top: 25px;">'
            +	     '<p id="btnCupConfirmLabel" style="display: table-cell; vertical-align: middle; padding: 0 20px; color: #fff">Ok</p>'
            +	  '</div>'
        
            break;
        

        case POPUP.FIRST_COFFEE:
            content = ''
            +     '<img src="img/TS_hot_water_warning.png" alt="HOT WATER WARNING" style="height: 100%; position: absolute; left: 0; right: 0; margin: 0 auto;">';
        
            break;

        default:
            break;
    }

    return content;
}
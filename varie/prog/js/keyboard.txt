// **************************************************************************************
//
// K E Y B O A R D   L A Y O U T 
// https://keyshorts.com/blogs/blog/44712961-how-to-identify-laptop-keyboard-localization
//
// **************************************************************************************

var Keyboard = {
    elements: {
        main: null,
        langs: [],
        keysContainer: null,
        keys: [],
        textElement: null,
        root: ''
    },

    eventHandlers: {
        oninput: null,
        onclose: null
    },

    properties: {
        value: "",
        capsLock: 0,
        altGr: false,
        accent: false,
        carretPosition: 0
    },

    init: function init(layout, root) {
        var self = this;

        this._resetKeyboard();

        // Create main elements
        this.elements.main = document.createElement("div");
        this.elements.keysContainer = document.createElement("div");
        this.elements.root = this.elements.root || root;

        // Setup main elements
        this.elements.main.classList.add("keyboard", "keyboard--hidden");
        this.elements.main.dataset.rheaKeyboardElement = true;
        this.elements.keysContainer.classList.add("keyboard__keys");

        var textReference = '' +
            '<div class="text-reference">' +
                '<div class="txt">&nbsp;</div>' +
            '</div>';

        this.elements.keysContainer.appendChild(this._stringToHTML(textReference));
        this.elements.keysContainer.appendChild(this._createKeys(layout));

		var langs = Object.keys(charset);

        for (var i = 0; i < langs.length; i++) {
            var lang = langs[i];

            self.elements.langs.push(lang);
        }

        this.elements.keys = this.elements.keysContainer.querySelectorAll(".keyboard__key");

        // Add to DOM
        this.elements.main.appendChild(this.elements.keysContainer);

        var button = '';

        for (var i = 0; i < this.elements.langs.length; i++ ) {
            var lang = this.elements.langs[i];

            button += '<button data-value="' + lang + '">' + lang + '</button>';
        }


        var selection = '' +
            '<div class="layout-type-container">' + 
                '<label for="langs">Keyboard layout</label>' + 
                '<br>' +
                '<div data-root="' + (this.elements.root || "") + '" name="langs" id="langs">' +
                    button +
                '</div>'
            '</div>';

        this.elements.main.appendChild(this._stringToHTML(selection));
        
        var buttons = this.elements.main.querySelectorAll('#langs button')
        for (var i=0; i<buttons.length; i++) {
		buttons[i].onclick = function(e) {
		    self.init(e.target.dataset.value, e.target.dataset.root);
		};
        }

        document.body.appendChild(this.elements.main);
        document.addEventListener('click', function(e) {
            if (!self.elements.textElement) return;

            if (e.target.closest('[data-rhea-keyboard-element="true"]') ||
                self.elements.textElement.contains(e.target) || self.elements.textElement === e.target) 
            {
                return;
            }

            self.close();
        });

        // Automatically use keyboard for elements with .use-keyboard-input
        var editables = document.querySelectorAll(".use-keyboard-input");

        for (var i = 0; i < editables.length; i++) {
            var element = editables[i];

            element.addEventListener("focus", function(event) {
                var _element = event.target;
                self.elements.textElement = _element;

                self.open(_element.value, function(currentValue) {
                    if (_element.type === 'number') {
                        if (currentValue.match(/[^$,.\d]/)) {
                            self.properties.carretPosition--;
                        }

                        currentValue = currentValue.replace(/\D/g, '');

                        _element.value = currentValue;
                        self.properties.value = currentValue;
                    } else {
                        _element.value = currentValue;
                    }
                });
            });
        }

        // Automatically use keyboard for elements with .use-keyboard-input
        for (var i = 0; i < editables.length; i++) {
            var element = editables[i];
            element.addEventListener("click", function(event) {
                self.properties.carretPosition = event.target.selectionStart || event.target.value.length;
            });
        };
    },

    /**
     * Convert a template string into HTML DOM nodes
     * @param  {String} str The template string
     * @return {Node}       The template HTML
     */
    _stringToHTML: function _stringToHTML(str) {
        var temp = document.createElement('div');
        temp.innerHTML = str;
        return temp;
    },

    _resetKeyboard: function _resetKeyboard() {
        this.elements = {
            main: null,
            langs: [],
            keysContainer: null,
            keys: [],
            textElement: null
        };

        this.properties = {
            value: "",
            capsLock: 0,
            altGr: false,
            accent: false,
            carretPosition: 0
        };

        if (document.querySelector('.keyboard')) {
            document.querySelector('.keyboard').remove();
        }
    },

    _createKeys: function _createKeys(layout) {
        var fragment = document.createDocumentFragment();
        var keyLayout = charset[layout]
        var self = this;
	
		self.keyLayout = keyLayout;
		
        for (var i = 0; i < self.keyLayout.length; i++) {
            var key = self.keyLayout[i];

            var keyElement = document.createElement("button");

            if (!key.newline) {
                // Add attributes/classes
                keyElement.setAttribute("type", "button");
                keyElement.classList.add("keyboard__key");

                switch (key.special) {
                    case "BACKSPACE":
                        keyElement.classList.add("keyboard__key");
                        keyElement.classList.add("backspace");

                        keyElement.dataset.special = true;

                        keyElement.addEventListener("click", function() {
                            var value = self.properties.value.substring(0, self.properties.carretPosition - 1) + self.properties.value.substring(self.properties.carretPosition, self.properties.value.length);

                            self.properties.carretPosition = self.properties.carretPosition !== 0 ? self.properties.carretPosition - 1 : 0;
                            self.properties.value = value;

                            self._triggerEvent("oninput");
                        });

                        break;

                    case "SHIFT":
                        keyElement.classList.add("keyboard__key", "keyboard__key--activatable");
                        keyElement.classList.add("shift");

                        keyElement.dataset.special = true;

                        keyElement.addEventListener("click", function() {
                            self.properties.capsLock = (self.properties.capsLock + 1) % 3;

                            self._toggleCapsLock();

                            keyElement.classList.toggle("keyboard__key--active", self.properties.capsLock === 2);
                        });

                        break;

                    case "ENTER":
                        keyElement.classList.add("keyboard__key");
                        keyElement.classList.add("enter");

                        keyElement.dataset.special = true;

                        keyElement.addEventListener("click", function() {
                            if (self.elements.textElement.nodeName === "INPUT") {
                                self.close();
                            }

                            var value = self.properties.value.substring(0, self.properties.carretPosition) + "\n" + self.properties.value.substring(self.properties.carretPosition)

                            self.properties.carretPosition++;
                            self.properties.value = value;

                            self._triggerEvent("oninput");
                        });

                        break;

                    case "ALTGR":
                        keyElement.classList.add("keyboard__key--wide", "keyboard__key--activatable");
                        keyElement.innerHTML = "Alt Gr";
                        keyElement.dataset.keyAltGr = true;

                        keyElement.dataset.special = true;

                        keyElement.addEventListener("click", function() {
                            self._toggleAltGr();

                            var accent = document.querySelector('[data-key-accent="true"]');

                            if (accent) {
                                accent.disabled = self.properties.altGr;
                            }

                            keyElement.classList.toggle("keyboard__key--active", self.properties.altGr);
                        });

                        break;

                    case "FLAG":
                    	var imgPath = (self.elements.root || "") + 'img/flags/' + key.cls.toUpperCase() + '.png';
                        keyElement.classList.add("keyboard__key");
                        keyElement.innerHTML = '<div class="key__flag" style="background-image: url(' + imgPath + ')"></div>';

                        keyElement.dataset.special = true;

                        keyElement.addEventListener("click", function(e) {
                            var selection = document.querySelector('.layout-type-container');
                            if (selection.style.display === 'block') {
                                selection.style.display = 'none';
                                return;
                            }

                            selection.style.display = 'block';

                            selection.style.left = (keyElement.offsetLeft - 100) + 'px'
                        });

                        break;

                    case "ACCENT":
                        keyElement.classList.add("keyboard__key", "keyboard__key--activatable");

                        keyElement.dataset.special = true;
                        keyElement.dataset.keyAccent = true;

                        keyElement.innerHTML = '' +
                            '<div class="key__container">' +
                                '<div class="key__sup disabled">' + key.sup + '</div>' +
                                '<div class="key__sub disabled">' + key.sub + '</div>' +
                            '</div>';

                        keyElement.addEventListener("click", function() {
                            self._toggleAccent();

                            var altGr = document.querySelector('[data-key-alt-gr="true"]');

                            if (altGr) {
                                altGr.disabled = self.properties.accent !== 0;
                            }

                            if (self.properties.accent === 1) {
                                keyElement.querySelector('.key__sub').classList.remove("disabled");
                            }

                            if (self.properties.accent === 2) {
                                keyElement.querySelector('.key__sub').classList.add("disabled");
                                keyElement.querySelector('.key__sup').classList.remove("disabled");
                            }

                            if (self.properties.accent === 0) {
                                keyElement.querySelector('.key__sub').classList.add("disabled");
                                keyElement.querySelector('.key__sup').classList.add("disabled");
                            }

                            keyElement.classList.toggle("keyboard__key--active", self.properties.accent);
                        });

                        break;

                    case "SPACE":
                        keyElement.classList.add("keyboard__key--extra-wide");

                        keyElement.dataset.special = true;

                        keyElement.addEventListener("click", function() {
                            var SPACE = " ";
                            var value = self.properties.value.substring(0, self.properties.carretPosition) + SPACE + self.properties.value.substring(self.properties.carretPosition)

                            self.properties.carretPosition++;
                            self.properties.value = value;

                            self._triggerEvent("oninput");
                        });

                        break;

                    case "CONFIRM":
                        keyElement.classList.add("keyboard__key--wide", "keyboard__key--dark");
                        keyElement.classList.add("check");

                        keyElement.dataset.special = true;

                        keyElement.addEventListener("click", function() {
                            self.close();
                            self._triggerEvent("onclose");
                        });

                        break;

                    default:
                        var html = '';

                        if (key.sup && key.altgr && key.altgrsup) {
                            html = '' +
                                '<div data-has-accent="${key.accent ? true : false}" class="key__container left">' +
                                    '<div data-index="' + i + '" class="key__sup disabled">' + key.sup +'</div>' +
                                    '<div data-index="' + i + '" class="key__sup keyboard__key__altgrsup altgr">' + key.altgrsup + '</div>' +
                                    '<div data-index="' + i + '" class="key__sub">' + key.sub + '</div>' +
                                    '<div data-index="' + i + '" class="keyboard__key__altgr altgr">' + key.altgr + '</div>' +
                                '</div>';

                        } else if (key.sup && key.altgr) {
                            html = '' +
                                '<div data-has-accent="' + (key.accent ? true : false) + '" class="key__container left">' +
                                    '<div data-index="' + i + '" class="key__sup disabled">' + key.sup + '</div>' +
                                    '<div data-index="' + i + '" class="key__sub">' + key.sub + '</div>' +
                                    '<div data-index="' + i + '" class="keyboard__key__altgr altgr">' + key.altgr + '</div>' +
                                '</div>';
                        } else if (key.sup) {
                            html = '' +
                                '<div data-has-accent="' + (key.accent ? true : false) + '" class="key__container">' +
                                    '<div data-index="' + i + '" class="key__sup disabled">' + key.sup + '</div>' +
                                    '<div data-index="' + i + '" class="key__sub">' + key.sub + '</div>' +
                                '</div>';
                        } else {
                            html = '' +
                                '<div data-has-accent="' + (key.accent ? true : false) + '" class="key__container">' +
                                    '<div data-index="' + i + '" class="std">' + key.sub + '</div>' +
                                    '<div data-index="' + i + '" class="keyboard__key__altgr altgr">' + (key.altgr || '') + '</div>' +
                                '</div>'
                        }

                        keyElement.innerHTML = html;

                        keyElement.addEventListener("click", function(event) {
                            var elem = event.target;
                            var _key = self.keyLayout[parseInt(elem.dataset.index)];

                            if (self.properties.altGr) {
                                if (_key.altgrsup && self.properties.capsLock) {
                                    var fn = self.properties.capsLock ? 'toUpperCase' : 'toLowerCase';
                                    var value = self.properties.value.substring(0, self.properties.carretPosition) + _key.altgrsup[fn]() + self.properties.value.substring(self.properties.carretPosition)

                                    self.properties.carretPosition++;
                                    self.properties.value = value;
                                } else if (_key.altgr) {
                                    var fn = self.properties.capsLock ? 'toUpperCase' : 'toLowerCase';
                                    var value = self.properties.value.substring(0, self.properties.carretPosition) + _key.altgr[fn]() + self.properties.value.substring(self.properties.carretPosition)

                                    self.properties.carretPosition++;
                                    self.properties.value = value;
                                }
                            } else if (self.properties.accent) {
                                if (_key.accent) {
                                    var fn = self.properties.capsLock ? 'toUpperCase' : 'toLowerCase';
                                    var character = self.properties.accent === 1 ? _key.a_1[fn]() : _key.a_2[fn]();
                                    var value = self.properties.value.substring(0, self.properties.carretPosition) + character + self.properties.value.substring(self.properties.carretPosition)

                                    self.properties.carretPosition++;
                                    self.properties.value = value;
                                }
                            } else {
                                if (self.properties.capsLock && !_key.sup) {
                                    var value = self.properties.value.substring(0, self.properties.carretPosition) + _key.sub.toUpperCase() + self.properties.value.substring(self.properties.carretPosition)

                                    self.properties.carretPosition++;
                                    self.properties.value = value;
                                } else if (self.properties.capsLock && _key.sup) {
                                    var value = self.properties.value.substring(0, self.properties.carretPosition) + _key.sup + self.properties.value.substring(self.properties.carretPosition)

                                    self.properties.carretPosition++;
                                    self.properties.value = value;
                                } else {
                                    var value = self.properties.value.substring(0, self.properties.carretPosition) + _key.sub + self.properties.value.substring(self.properties.carretPosition)

                                    self.properties.carretPosition++;
                                    self.properties.value = value;
                                }
                            }

                            if (self.properties.capsLock === 1)
                                self.properties.capsLock = 0;

                            self._toggleCapsLock();

                            self._triggerEvent("oninput");
                        });

                        break;
                }

                fragment.appendChild(keyElement);
            }

            if (key.newline) {
                fragment.appendChild(document.createElement("br"));
            }
        };

        return fragment;
    },

    _triggerEvent: function _triggerEvent(handlerName) {
        if (typeof this.eventHandlers[handlerName] == "function") {
            this.eventHandlers[handlerName](this.properties.value);

            if (handlerName === 'oninput') {
                this.elements.main.querySelector('.text-reference .txt').innerHTML = this.properties.value;
            }
        }
    },

    _toggleAltGr: function _toggleAltGr() {
        this.properties.altGr = !this.properties.altGr;

        for (var i = 0; i < this.elements.keys.length; i++) {
            var key = this.elements.keys[i];

            if (key.querySelector('.key__sup')) {
                key.querySelector('.key__sup').classList.toggle("altgr");
            }

            if (key.querySelector('.key__sub')) {
                key.querySelector('.key__sub').classList.toggle("altgr");
            }

            if (key.querySelector('.std')) {
                key.querySelector('.std').classList.toggle("altgr");
            }

            if (key.querySelector('.keyboard__key__altgr')) {
                if (!this.properties.capsLock && this.properties.altGr) key.querySelector('.keyboard__key__altgr').classList.remove("altgr");
                if (!this.properties.capsLock && this.properties.altGr) {
                    if (key.querySelector('.keyboard__key__altgrsup')) key.querySelector('.keyboard__key__altgrsup').classList.add("altgr");
                }
            }

            if (key.querySelector('.keyboard__key__altgrsup')) {
                if (this.properties.capsLock && this.properties.altGr) {
                    if (key.querySelector('.keyboard__key__altgr')) key.querySelector('.keyboard__key__altgr').classList.add("altgr");
                }
                if (this.properties.capsLock && this.properties.altGr) key.querySelector('.keyboard__key__altgrsup').classList.remove("altgr");
            }

            if (!this.properties.altGr) {
                if (key.querySelector('.keyboard__key__altgr')) key.querySelector('.keyboard__key__altgr').classList.add("altgr");
                if (key.querySelector('.keyboard__key__altgrsup')) key.querySelector('.keyboard__key__altgrsup').classList.add("altgr");
            }
        }
    },

    _toggleCapsLock: function _toggleCapsLock() {
        for (var i = 0; i < this.elements.keys.length; i++) {
            var key = this.elements.keys[i];

            if (key.dataset.special) {
                continue;
            }

            if (key.querySelector('.std')) {
                key.querySelector('.std').innerHTML = this.properties.capsLock > 0 ? key.querySelector('.std').innerHTML.toUpperCase() : key.querySelector('.std').innerHTML.toLowerCase();
            }

            if (this.properties.capsLock > 0) {
                if (key.querySelector('.key__sup')) key.querySelector('.key__sup').classList.remove("disabled");
                if (key.querySelector('.key__sub')) key.querySelector('.key__sub').classList.add("disabled");
            }

            if (this.properties.capsLock === 0) {
                if (key.querySelector('.key__sup')) key.querySelector('.key__sup').classList.add("disabled");
                if (key.querySelector('.key__sub')) key.querySelector('.key__sub').classList.remove("disabled");
            }


            if (key.querySelector('.keyboard__key__altgr')) {
                if (!this.properties.capsLock && this.properties.altGr) key.querySelector('.keyboard__key__altgr').classList.remove("altgr");
                if (!this.properties.capsLock && this.properties.altGr) {
                    if (key.querySelector('.keyboard__key__altgrsup')) key.querySelector('.keyboard__key__altgrsup').classList.add("altgr");
                }
            }

            if (key.querySelector('.keyboard__key__altgrsup')) {
                if (this.properties.capsLock && this.properties.altGr) {
                    if (key.querySelector('.keyboard__key__altgr')) key.querySelector('.keyboard__key__altgr').classList.add("altgr");
                }
                if (this.properties.capsLock && this.properties.altGr) key.querySelector('.keyboard__key__altgrsup').classList.remove("altgr");
            }

            if (!this.properties.altGr) {
                if (key.querySelector('.keyboard__key__altgr')) key.querySelector('.keyboard__key__altgr').classList.add("altgr");
                if (key.querySelector('.keyboard__key__altgrsup')) key.querySelector('.keyboard__key__altgrsup').classList.add("altgr");
            }
        }
    },

    _toggleAccent: function _toggleAccent() {
        this.properties.accent = (this.properties.accent + 1) % 3;

        var elements = document.querySelectorAll('[data-has-accent="false"]');

        for (var i = 0; i < elements.length; i++) {
            var key = elements[i];
            key.classList[this.properties.accent ? 'add' : 'remove']('accent');
        }
    },

    open: function open(initialValue, oninput, onclose) {
        this.properties.value = initialValue || "";
        this.eventHandlers.oninput = oninput;
        this.eventHandlers.onclose = onclose;
        this.elements.main.classList.remove("keyboard--hidden");

        this.elements.main.querySelector('.text-reference .txt').innerHTML = this.properties.value;
    },

    close: function close() {
        this.properties.value = "";
        this.eventHandlers.oninput = null;
        this.eventHandlers.onclose = null;
        this.elements.main.classList.add("keyboard--hidden");

        this.elements.main.querySelector('.text-reference .txt').innerHTML = "";
    }
};

var charset = {
    EN: [
        { sub: "`", sup: "¬", altgr: "¦" },
        { sub: "1", sup: "!", altgr: null },
        { sub: "2", sup: '"', altgr: null },
        { sub: "3", sup: "£", altgr: null },
        { sub: "4", sup: "$", altgr: "€" },
        { sub: "5", sup: "%", altgr: null },
        { sub: "6", sup: "^", altgr: null },
        { sub: "7", sup: "&", altgr: null },
        { sub: "8", sup: "*", altgr: null },
        { sub: "9", sup: "(", altgr: null },
        { sub: "0", sup: ")", altgr: null },
        { sub: "-", sup: "_", altgr: null },
        { sub: "=", sup: "+", altgr: null },
        { special: 'BACKSPACE' },
        { newline: true },
        // --------------------------------
        { sub: "q", sup: null, altgr: null },
        { sub: "w", sup: null, altgr: null },
        { sub: "e", sup: null, altgr: null },
        { sub: "r", sup: null, altgr: null },
        { sub: "t", sup: null, altgr: null },
        { sub: "y", sup: null, altgr: null },
        { sub: "u", sup: null, altgr: null },
        { sub: "i", sup: null, altgr: null },
        { sub: "o", sup: null, altgr: null },
        { sub: "p", sup: null, altgr: null },
        { sub: "[", sup: "{", altgr: null },
        { sub: "]", sup: "}", altgr: null },
        { special: 'ENTER' },
        { newline: true },
        // ---------------------------------
        { special: 'SHIFT' },
        { sub: "a", sup: null, altgr: null },
        { sub: "s", sup: null, altgr: null },
        { sub: "d", sup: null, altgr: null },
        { sub: "f", sup: null, altgr: null },
        { sub: "g", sup: null, altgr: null },
        { sub: "h", sup: null, altgr: null },
        { sub: "j", sup: null, altgr: null },
        { sub: "k", sup: null, altgr: null },
        { sub: "l", sup: null, altgr: null },
        { sub: ";", sup: ":", altgr: null },
        { sub: "'", sup: "@", altgr: null },
        { sub: "#", sup: "~", altgr: null },
        { newline: true },
        // ---------------------------------
        { sub: "\\", sup: "|", altgr: null },
        { sub: "z", sup: null, altgr: null },
        { sub: "x", sup: null, altgr: null },
        { sub: "c", sup: null, altgr: null },
        { sub: "v", sup: null, altgr: null },
        { sub: "b", sup: null, altgr: null },
        { sub: "n", sup: null, altgr: null },
        { sub: "m", sup: null, altgr: null },
        { sub: ",", sup: "<", altgr: null },
        { sub: ".", sup: ">", altgr: null },
        { sub: "/", sup: "?", altgr: null },
        { newline: true },
        // ---------------------------------
        { special: 'CONFIRM' },
        { special: 'SPACE' },
        { special: 'ALTGR' },
        { special: 'FLAG', cls: 'EN' },
    ],
    IT: [
        { sub: "\\", sup: "|" },
        { sub: "1", sup: "!" },
        { sub: "2", sup: '"' },
        { sub: "3", sup: "£" },
        { sub: "4", sup: "$" },
        { sub: "5", sup: "%", altgr: "€" },
        { sub: "6", sup: "&" },
        { sub: "7", sup: "/" },
        { sub: "8", sup: "(" },
        { sub: "9", sup: ")" },
        { sub: "0", sup: "=" },
        { sub: "'", sup: "?" },
        { sub: "ì", sup: "^" },
        { special: 'BACKSPACE' },
        { newline: true },
        // --------------------------------
        { sub: "q", sup: null, altgr: null },
        { sub: "w", sup: null, altgr: null },
        { sub: "e", sup: null, altgr: "€" },
        { sub: "r", sup: null, altgr: null },
        { sub: "t", sup: null, altgr: null },
        { sub: "y", sup: null, altgr: null },
        { sub: "u", sup: null, altgr: null },
        { sub: "i", sup: null, altgr: null },
        { sub: "o", sup: null, altgr: null },
        { sub: "p", sup: null, altgr: null },
        { sub: "è", sup: "é", altgr: "[", altgrsup: '{' },
        { sub: "+", sup: "*", altgr: "]", altgrsup: '}' },
        { special: 'ENTER' },
        { newline: true },
        // ---------------------------------
        { special: 'SHIFT' },
        { sub: "a", sup: null, altgr: null },
        { sub: "s", sup: null, altgr: null },
        { sub: "d", sup: null, altgr: null },
        { sub: "f", sup: null, altgr: null },
        { sub: "g", sup: null, altgr: null },
        { sub: "h", sup: null, altgr: null },
        { sub: "j", sup: null, altgr: null },
        { sub: "k", sup: null, altgr: null },
        { sub: "l", sup: null, altgr: null },
        { sub: "ò", sup: "ç", altgr: "@" },
        { sub: "à", sup: "°", altgr: "#" },
        { sub: "ù", sup: "§", altgr: null },
        { newline: true },
        // ---------------------------------
        { sub: "<", sup: ">", altgr: null },
        { sub: "z", sup: null, altgr: null },
        { sub: "x", sup: null, altgr: null },
        { sub: "c", sup: null, altgr: null },
        { sub: "v", sup: null, altgr: null },
        { sub: "b", sup: null, altgr: null },
        { sub: "n", sup: null, altgr: null },
        { sub: "m", sup: null, altgr: null },
        { sub: ",", sup: ";", altgr: null },
        { sub: ".", sup: ":", altgr: null },
        { sub: "-", sup: "_", altgr: null },
        { newline: true },
        // ---------------------------------
        { special: 'CONFIRM' },
        { special: 'SPACE' },
        { special: 'ALTGR' },
        { special: 'FLAG', cls: 'IT' },
    ],
    FR: [
        { sub: "&", sup: "1" },
        { sub: "é", sup: "2", altgr: "~" },
        { sub: '"', sup: "3", altgr: "#" },
        { sub: "'", sup: "4", altgr: "{" },
        { sub: "(", sup: "5", altgr: "[" },
        { sub: "-", sup: "6", altgr: "|" },
        { sub: "è", sup: "7", altgr: "`" },
        { sub: "_", sup: "8", altgr: "\\" },
        { sub: "ç", sup: "9", altgr: "^" },
        { sub: "à", sup: "0", altgr: "@" },
        { sub: ")", sup: "°", altgr: "]" },
        { sub: "=", sup: "+", altgr: "}" },
        { special: 'BACKSPACE' },
        { newline: true },
        // --------------------------------
        { sub: "a", sup: null, altgr: "æ", accent: true, a_1: "â", a_2: "ä" },
        { sub: "z", sup: null, altgr: null },
        { sub: "e", sup: null, altgr: "€", accent: true, a_1: "ê", a_2: "ë" },
        { sub: "r", sup: null, altgr: null },
        { sub: "t", sup: null, altgr: null },
        { sub: "y", sup: null, altgr: null },
        { sub: "u", sup: null, altgr: null, accent: true, a_1: "û", a_2: "ü" },
        { sub: "i", sup: null, altgr: null, accent: true, a_1: "î", a_2: "ï" },
        { sub: "o", sup: null, altgr: "œ", accent: true, a_1: "ô", a_2: "ö" },
        { sub: "p", sup: null, altgr: null },
        { sub: "£", sup: "$", altgr: null },
        { special: 'ENTER' },
        { newline: true },
        // ---------------------------------
        { special: 'SHIFT' },
        { sub: "q", sup: null, altgr: null },
        { sub: "s", sup: null, altgr: null },
        { sub: "d", sup: null, altgr: null },
        { sub: "f", sup: null, altgr: null },
        { sub: "g", sup: null, altgr: null },
        { sub: "h", sup: null, altgr: null },
        { sub: "j", sup: null, altgr: null },
        { sub: "k", sup: null, altgr: null },
        { sub: "l", sup: null, altgr: null },
        { sub: "m", sup: null, altgr: null },
        { sub: "ù", sup: "%", altgr: null },
        { sub: "*", sup: "µ", altgr: null },
        { newline: true },
        // ---------------------------------
        { sub: "<", sup: ">", altgr: null },
        { sub: "w", sup: null, altgr: null },
        { sub: "x", sup: null, altgr: null },
        { sub: "c", sup: null, altgr: null },
        { sub: "v", sup: null, altgr: null },
        { sub: "b", sup: null, altgr: null },
        { sub: "n", sup: null, altgr: null },
        { sub: ",", sup: "?", altgr: null },
        { sub: ";", sup: ".", altgr: null },
        { sub: ":", sup: "/", altgr: null },
        { sub: "ù", sup: "%", altgr: null },
        { sub: "!", sup: "§", altgr: null },
        { newline: true },
        // ---------------------------------
        { special: 'CONFIRM' },
        { special: 'SPACE' },
        { special: 'ALTGR' },
        { special: 'ACCENT', sub: "^", sup: "¨", altgr: null },
        { special: 'FLAG', cls: 'FR' },
    ]
}

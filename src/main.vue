<template> 
	<select ref="select"><slot/></select>
</template> 

<script>

/*
 * slideToggle <bool> - If its true the options will slide
 * disableTriggerOnChange <function> - If its called the onChange event will not called anymore
 * enableTriggerOnChange <function> - If its called the onChange event will callend again
 * onItemRemove <function(items, removedItem)> - This function called when an item removed
 * setFocus <function>
 * setBlur <function>
 * setValue <function (value)>
 * setOptions <function (array or object of option)>
 * addOptions <function (array or object of option)>
 * addOption <function (object of option)>
 * setItems <function (object of items, if its true its added when option is not exists)> - Remove all items and add
 * addItems <function (object of items, if its true its added when option is not exists)> - Just add items, not remove
 * addItem <function (item, if its true its added when option is not exists)>
 * addOptionsIfNotExists <function (options)>
 * addOptionIfNotExists <function (option)>
 * addItemAsOption <function (option)> - Add item (param will option not item) Option has value and text field
 * createOnEnter <bool> - Create will run when you press enter and text is not empty
 * createOnBlur <bool> - Create will run when you click outside and text is not empty
 * debug <bool> - Enable debug mode

 * inputText - Text in input
 * element - Element of select
 * options - Options array
 * focus - If focused
 */

import $ from 'jquery';
import equal from 'deep-equal';

if (!$().selectize) {
	require('selectize')
}

function clean(options) {
	return options.map(option => ({
		text: option.text,
		value: option.value
	}));
}

export default {
	props: {
		value: {
			default: ''
		},
		settings: {
			type: Object,
			default: () => ({
				labelField: 'text',
				valueField: 'value'
			})
		},
		disabled: {
			type: Boolean,
			default: false
		},
		options: {
			type: Array
		}
	},
	data() {
		return {
			element: {},
			selectize: {},
			currentOptions: [],
			createdOptions: [],
			focus: false,
			inputText: ''
		}
	},
	mounted() {
		var self = this;

		this.element = this.$refs.select;

		// If create is bool
		if (this.settings.create) {
			var create = this.settings.create;
			this.settings.create = function(input, callback) {
				self.log('Create: ' + input);
				var option = null
				if (create === true) {
					option = {
						text: input,
						value: input
					}
				} else {
					option = create(input, callback, self);
				}
				self.createdOptions.push(option);
				return option;
			}

		}

		// Slide toggle
		if (this.settings.slideToggle) {
			var onDropdownOpen = this.settings.onDropdownOpen;
			var onDropdownClose = this.settings.onDropdownClose;
			this.settings.onDropdownOpen = function($dropdown = null) {
				$(this.$dropdown).hide().slideDown('fast').fadeIn('fast');
				if (onDropdownOpen) onDropdownOpen($dropdown);
			}
			this.settings.onDropdownClose = function($dropdown = null) {
				$(this.$dropdown).show().slideUp('fast').fadeOut('fast');
				if (onDropdownClose) onDropdownClose($dropdown);
			}
		}

		// Init selectize
		$(this.element).selectize({
			onInitialize: function() {
				self.selectize = this;
				self.setValue();
			},
			onChange: value => {
				this.$emit('input', value);
				if (this.settings.onChange) this.settings.onChange(value);
			},
			onFocus: param => {
				this.focus = true;
				if (this.settings.onFocus) this.settings.onFocus(param);
			},
			onBlur: param => {
				this.focus = false;
				if (this.settings.onBlur) this.settings.onBlur(param);
			},
			setOptions: this.setOptions,
			addOptions: this.addOptions,
			addOptionsIfNotExists: this.addOptionsIfNotExists,
			addOptionIfNotExists: this.addOptionIfNotExists,
			setItems: this.setItems,
			addItems: this.addItems,
			addItem: this.addItem,
			disableTriggerOnChange: this.disableTriggerOnChange,
			enableTriggerOnChange: this.enableTriggerOnChange,
			...this.settings
		})

		// At init
		this.makeOptions(true);
		this.toggleDisabled(this.disabled);

		$(this.element).find('input').on('input', e => {
			this.inputText = e.target.value;

			// Call create on enter
			if (this.inputText.length && this.settings.createOnEnter && e.keyCode === 13 && this.focus && this.settings.create) {
				e.preventDefault();
				this.settings.create(this.inputText, () => {
					this.addItem(this.inputText, true);
					this.log('Item added: ' + this.inputText);
				});
				this.log('Add item: ' + this.inputText);
			}
		});
	},
	destroyed() {
		if (this.element.selectize) {
			this.element.selectize.destroy();
		}
	},
	watch: {
		value(value, old) {
			if (!equal(value, old)) {
				this.setValue();
			}

			// Call onItemRemove
			if (this.settings.onItemRemove && Array.isArray(value) && Array.isArray(old) && value.length < old.length) {
				var removedItem = old.filter(e => !value.find(a => e == a));
				this.settings.onItemRemove(value, removedItem);
				this.log('On item remove');
			}
		},
		disabled(disabled) {
			this.toggleDisabled(disabled)
		},
		focus(focus) {
			if (focus === false) {

				// Call create on blur
				if (this.inputText.length && this.settings.createOnBlur && this.settings.create) {
					this.settings.create(this.inputText, () => {
						this.addItem(this.inputText, true);
						this.log('Item added: ' + this.inputText);
					});
					this.log('Add item: ' + this.inputText);
				}
			}
		},

		// Options from props
		options(options) {
			this.setOptions(options);
		}
	},
	methods: {
		toggleDisabled(value) {
			if (value) {
				this.element.selectize.disable();
			} else {
				this.element.selectize.enable();
			}
		},
		makeOptions(justLocal = false) {
			var old = this.currentOptions
			let _new = []
			var nodes = this.$slots.default
			if (this.settings.options === undefined && nodes) {
				_new = nodes.filter(node => node.tag && node.tag.toLowerCase() === 'option').map(node => {
					return {
						text: node.children ? node.children[0].text.trim() : null,
						value: node.data.domProps ? node.data.domProps.value : node.data.attrs.value
					}
				}).concat(this.createdOptions)
			}
			if (!equal(clean(old), clean(_new))) {
				this.currentOptions = _new
				if (!justLocal) {
					this.element.selectize.clearOptions();
					var optionValues = this.currentOptions.map(o => o.value)
					Object.keys(this.element.selectize.options)
						//IE11 fix, Object.values is not supported
						.map(key => this.element.selectize.options[key]).filter(option => optionValues.every(v => !equal(v, option.value))).forEach(option => this.element.selectize.removeOption(option.value));
					this.element.selectize.addOption(this.currentOptions);
					this.element.selectize.refreshOptions(false);
					this.setValue();
				}
			}
		},
		setValue(value) {
			if (!value) value = this.value;
			if (this.settings.forceAdding) {
				this.addOptionsIfNotExists(value);
			}
			this.element.selectize.setValue(value, true);
			this.log('Value Set: ' + value);
		},
		setOptions(options) {
			// Save selected items before clear options (like backup)
			var items = this.value;

			// Disable onchange event while items readding
			this.disableTriggerOnChange();

			// Add options, clearOptions will remove items too
			this.element.selectize.clearOptions();
			options.forEach(option => this.element.selectize.addOption(option));

			// Set items form backup
			this.addItems(items);

			this.element.selectize.refreshOptions(false)
			this.setValue();

			// Reload onchange event
			this.enableTriggerOnChange();
		},

		// Add options if array 
		addOptions(options) {

			if (Array.isArray(options)) {
				options.forEach(option => this.element.selectize.addOption(option));
				return options;
			}
			
			this.addOption(options);
			return options;
		},

		// Add one option
		addOption(option) {
			this.element.selectize.addOption(option);
			this.element.selectize.refreshOptions(false);
		},
		setItems(items, force = false) {

			// Disable onchange event while items readding
			this.disableTriggerOnChange();

			// Set items
			this.element.selectize.clearItems();
			this.addItems(items, force);

			// Reload onchange event
			this.enableTriggerOnChange();
		},
		addItems(items, force = false) {

			if (Array.isArray(items)) {
				items.forEach(item => this.addItem(item));
				return items;
			}

			this.addItem(items, force);
			return items;
		},
		addItem(item, force = false) {
			if (force) this.addOptionIfNotExists(item);
			this.element.selectize.addItem(items);
		},
		removeItem(item) {
			this.element.selectize.removeItem(item);
			this.setValue();
		},
		addOptionsIfNotExists(values) {
			values.forEach(value => this.addOptionIfNotExists(value));
		},
		addOptionIfNotExists(value) {
			var found = false;
			var valueField = this.settings.valueField || 'value';
			var labelField = this.settings.labelField || 'text';

			// Find by value
			this.currentOptions.forEach(function(option) {
				if (option[valueField] === value) {
					found = true;
					return;
				}
			});

			// If option not exists add
			if (found === true) return true;

			var option = {};
			option[valueField] = value;
			option[labelField] = value;
			this.element.selectize.addOption(option);
			return true;
		},
		addItemAsOption(option) {

			// Find option by valueField
			var valueField = this.settings.valueField || 'value';

			this.element.selectize.addOptionIfNotExists(option);
			this.element.selectize.addItem(option[valueField]);
			this.setValue();
		},
		setFocus() {
			this.element.selectize.focus();
		},
		setBlur() {
			this.element.selectize.blur();
		},
		log(text) {
			if (this.settings.debug) console.log('Selectize -- ' + text);
		},
		disableTriggerOnChange() {
			this.log('On Change disabled');
			this.element.selectize.onChange = function() {};
			this.oldOnChange = this.settings.onChange;
			this.triggerOnChange = false;
		},
		enableTriggerOnChange() {
			this.log('On Change enabled');
			this.element.selectize.onChange = this.oldOnChange;
			this.oldOnChange = function() {};
			this.triggerOnChange = true;
		}
	},
	beforeUpdate() {
		this.makeOptions();
	}
} 
</script>
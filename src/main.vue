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
 * createIfNotExists <bool> - If push something in the model and not exists
 * debug <bool> - Enable debug mode
 * disableItemRemove

 * inputText - Text in input
 * element - Element of select
 * options - Options array
 * focus - If focused
 */

import $ from 'jquery';
import equal from 'deep-equal';

if (!$().selectize) {
	require('selectize');
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
			inputText: '',
			items: [],
			options: [],
			lastOptions: []
		};
	},
	mounted() {
		let self = this;

		this.element = this.$refs.select;
		this.log('Element initialized', this.element);

		// If create is bool
		if (this.settings.create) {
			let create = this.settings.create;
			this.settings.create = function(input, callback) {
				self.log('Create: ' + input);
				let option = null;
				if (create === true) {
					option = {
						text: input,
						value: input
					};
				} else {
					option = create(input, callback, this);
				}
				self.createdOptions.push(option);
				return option;
			};
		}

		// Slide toggle
		if (this.settings.slideToggle) {
			let onDropdownOpen = this.settings.onDropdownOpen;
			let onDropdownClose = this.settings.onDropdownClose;
			this.settings.onDropdownOpen = function($dropdown = null) {

				if (this.settings.selectOnlyOneItem && this.items && this.items.length) {
					return;
				}

				if (self.element) {
					let dropdownElement = $(self.element).find('.selectize-dropdown');
					if (dropdownElement) {
						dropdownElement
							.hide()
							.slideDown('fast')
							.fadeIn('fast');
					}
				}
				if (onDropdownOpen) onDropdownOpen($dropdown);
			};
			this.settings.onDropdownClose = function($dropdown = null) {
				if (self.element) {
					let dropdownElement = $(self.element).find('.selectize-dropdown');
					if (dropdownElement) {
						dropdownElement
							.show()
							.slideUp('fast')
							.fadeOut('fast');
					}
				}
				if (onDropdownClose) onDropdownClose($dropdown);
			};
		}

		// If its true, the user cannot remove item
		if (this.settings.disableItemRemove) {
			let onItemRemove = this.settings.onItemRemove;
			this.settings.onItemRemove = function(value) {
				selectize.setItems(val);
				if (onItemRemove) onItemRemove(value);
			};
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
			selectOnlyOneItem: this.selectOnlyOneItem,
			...this.settings
		});

		// At init
		this.makeOptions(true);
		this.toggleDisabled(this.disabled);

		$(this.element)
			.find('input')
			.on('input', e => {
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
				let removedItem = old.filter(e => !value.find(a => e == a));
				this.settings.onItemRemove(value, removedItem);
				this.log('On item remove');
			}
		},
		disabled(disabled) {
			this.toggleDisabled(disabled);
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
			let old = this.currentOptions;
			let _new = [];
			let nodes = this.$slots.default;
			if (this.settings.options === undefined && nodes) {
				_new = nodes
					.filter(node => node.tag && node.tag.toLowerCase() === 'option')
					.map(node => {
						return {
							text: node.children ? node.children[0].text.trim() : null,
							value: node.data.domProps ? node.data.domProps.value : node.data.attrs.value
						};
					})
					.concat(this.createdOptions);
			}
			if (!equal(clean(old), clean(_new))) {
				this.currentOptions = _new;
				if (!justLocal) {
					this.element.selectize.clearOptions();
					let optionValues = this.currentOptions.map(o => o.value);
					Object.keys(this.element.selectize.options)
						//IE11 fix, Object.values is not supported
						.map(key => this.element.selectize.options[key])
						.filter(option => optionValues.every(v => !equal(v, option.value)))
						.forEach(option => this.element.selectize.removeOption(option.value));
					this.element.selectize.addOption(this.currentOptions);
					this.element.selectize.refreshOptions(false);
					this.setValue();
				}
			}
		},
		setValue(value, forceAdding = false) {
			if (!value) value = this.value;
			if (this.settings.forceAdding || forceAdding === true) {
				if (Array.isArray(value)) {
					this.addOptionsIfNotExists(value);
				} else {
					this.addOptionIfNotExists(value);
				}
			}
			this.element.selectize.setValue(value, true);
			this.log('Value Set: ' + value);
		},
		clearOptions() {
			return this.setOptions([]);
		},
		setOptions(options) {
			// Save selected items before clear options (like backup)
			let items = this.value;

			// Set options value
			this.currentOptions = options;

			// Disable onchange event while items readding
			this.disableTriggerOnChange();

			// Add options, clearOptions will remove items too
			this.element.selectize.clearOptions();
			options.forEach(option => this.element.selectize.addOption(option));

			// Set items form backup
			this.addItems(items);

			this.element.selectize.refreshOptions(false);
			this.setValue();

			// Set last options, if more than 1
			if (this.currentOptions && this.currentOptions.length) {
				this.lastOptions = this.currentOptions;
			}

			// Reload onchange event
			this.enableTriggerOnChange();
			return this.currentOptions;
		},

		// Add options if array
		addOptions(options) {
			if (Array.isArray(options)) {
				options.forEach(option => this.element.selectize.addOption(option));
				return options;
			}

			this.addOption(options);

			// Set last options, if more than 1
			if (this.currentOptions && this.currentOptions.length) {
				this.lastOptions = this.currentOptions;
			}

			return this.currentOptions;
		},

		// Add one option
		addOption(option) {
			let valueField = this.settings.valueField || 'value';
			let labelField = this.settings.labelField || 'text';
			let findOldOption = this.currentOptions.find(oldOption => oldOption[valueField] === option[valueField]);

			// If option is already added, remove it to overwrite
			if (findOldOption) {
				this.element.selectize.removeOption(findOldOption[valueField]);
			}

			this.element.selectize.addOption(option);
			this.element.selectize.refreshOptions(false);

			// Set last options, if more than 1
			if (this.currentOptions && this.currentOptions.length) {
				this.lastOptions = this.currentOptions;
			}

			return this.currentOptions;
		},
		setItems(items, force = false) {
			// Disable onchange event while items readding
			this.disableTriggerOnChange();

			// Set items
			this.element.selectize.clearItems();
			this.addItems(items, force);


			// Reload onchange event
			this.enableTriggerOnChange();

			return this.items;
		},
		addItems(items, force = false) {

			// If its array each in items
			if (Array.isArray(items)) {

				// Each in items and push it
				items.forEach(item => this.addItem(items));

				// Generated in addItem functon
				return this.items;
			}

			// If its object insert it
			this.addItem(items, force);


			return this.items;
		},
		addItem(value, force = false) {

			if (force) {
				this.addOptionIfNotExists(value);
			}

			// Search value from options
			value = this.getValueFromOptions(value);

			// Add item to selectize
			this.element.selectize.addItem(value);

			// Set component variable
			this.items = this.items.concat(value);

			return [value];
		},
		removeItem(value) {
			value = this.getValueFromOptions(value);
			this.element.selectize.removeItem(value);
			this.setValue();
			return value;
		},
		removeAllOptionsExcept(notRemovableOptions) {
			let valueField = this.settings.valueField || 'value';
			let labelField = this.settings.labelField || 'text';

			// Convert remove options to array
			notRemovableOptions = Array.isArray(notRemovableOptions) ? notRemovableOptions : [notRemovableOptions];

			var notRemovableOptionIds = notRemovableOptions.map(option => {

				// If option is object, get valueField
				if (typeof option === 'object' && !Array.isArray(option)) {
					return option[valueField];
				}

				// If its string or int or bool
				return option;

			});

			// Filter options, if removeIds not found in options
			var optionsAfterFilter = this.currentOptions.filter(option => optionToRemoveIds.indexOf(option[valueField]) === -1);
			

			// Set filtered options
			this.setOptions(optionsAfterFilter);

			return this.currentOptions;

		},
		removeOptions(removeOptions) {
			let valueField = this.settings.valueField || 'value';
			let labelField = this.settings.labelField || 'text';

			// Convert remove options to array
			removeOptions = Array.isArray(removeOptions) ? removeOptions : [removeOptions];

			var optionToRemoveIds = removeOptions.map(option => {

				// If option is object, get valueField
				if (typeof option === 'object' && !Array.isArray(option)) {
					return option[valueField];
				}

				// If its string or int or bool
				return option;

			});

			// Filter options
			var optionsAfterFilter = this.currentOptions.filter(option => optionToRemoveIds.indexOf(option[valueField]) !== -1);

			// Set filtered options
			this.setOptions(optionsAfterFilter);

			return this.currentOptions;
		},
		addOptionsIfNotExists(values) {

			if (Array.isArray(value)) {
				values.forEach(value => this.addOptionIfNotExists(value));
				return this.currentOptions;
			}

			return this.addOptionIfNotExists(values);
		},
		addOptionIfNotExists(value) {
			let found = false;
			let valueField = this.settings.valueField || 'value';
			let labelField = this.settings.labelField || 'text';

			// Check value is not empty
			if (!value && !value.trim()) {
				this.log('Value is empty when adding option');
				return;
			}

			// Find by value
			this.currentOptions.forEach(function(option) {
				if (option[valueField] === value) {
					this.log('Value is already added to options');
					found = true;
					return;
				}
			});

			// If option not exists add
			if (found === true) return value;

			// Set option values
			let option = {};
			option[valueField] = value;
			option[labelField] = value;

			// Add option to selectize
			this.element.selectize.addOption(option);

			return this.currentOptions;
		},
		addItemAsOption(option) {
			// Find option by valueField
			let valueField = this.settings.valueField || 'value';

			// Add option
			this.addOptionIfNotExists(option);

			// Add item to selectize
			this.element.selectize.addItem(option[valueField]);

			this.setValue();

			return this.currentOptions;
		},
		setFocus() {
			this.element.selectize.focus();
		},
		setBlur() {
			this.element.selectize.blur();
		},
		log(text, text2 = '', text3 = '') {
			if (this.settings.debug) console.log('Selectize -- ', text, text2, text3);
		},
		disableTriggerOnChange() {
			this.log('On Change disabled');
			if (this.element.selectize) this.element.selectize.onChange = function() {};
			this.oldOnChange = this.settings.onChange;
			this.triggerOnChange = false;
		},
		enableTriggerOnChange() {
			this.log('On Change enabled');
			if (this.element.selectize) this.element.selectize.onChange = this.oldOnChange;
			this.oldOnChange = function() {};
			this.triggerOnChange = true;
		},
		// As value you can push a string and option object
		// If its string check is in options
		// If its object get the value
		getValueFromOptions(value) {
			// Check value is an object
			let valueField = this.settings.valueField || 'value';
			if (!Array.isArray(value)) {
				// Check value field is exists
				if (!value[valueField]) {
					this.log('Item is object, but ' + valueField + ' field is not exists in ' + JSON.stringify(value));
				}

				return value[valueField];
			}

			// Check option is exists
			if (!this.currentOptions.find(option => option[valueField] == value)) {
				this.log('Item not exists in options with value ' + value);
				return value;
			}

			return value;
		},

		setOptionGroups(optgroups) {
			var self = this;
			var optgroupId = self.settings.optgroupValueField || 'id';
			if (Array.isArray(optgroups)) {
				self.selectize.clearOptionGroups();
				optgroups.forEach(function(optgroup) {
					self.selectize.addOptionGroup(optgroup[optgroupId], optgroup);
				});
			}
		}
	},
	beforeUpdate() {
		this.makeOptions();
	}
};
</script>

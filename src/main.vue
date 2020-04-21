<template> 
	<select><slot/></select>
</template> 

<script>
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
		}
	},
	data() {
		return {
			selectize: {},
			options: [],
			createdOptions: [],
			focus: false,
			inputText: ''
		}
	},
	mounted() {
		var self = this

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
				onDropdownOpen($dropdown);
			}
			this.settings.onDropdownClose = function($dropdown = null) {
				$(this.$dropdown).show().slideUp('fast').fadeOut('fast');
				onDropdownClose($dropdown);
			}
		}

		// Flip
		if (this.settings.flipToggle) {			
			var onDropdownOpen = this.settings.onDropdownOpen;
			var onDropdownClose = this.settings.onDropdownClose;
			this.settings.onDropdownOpen = function($dropdown = null) {
				$(this.$dropdown).hide().css({
					'position': 'absolute',
					'transition': 'all 0.2s linear',
					'transform-origin': '0 0',
					'-webkit-transform': 'rotateX(90)',
					'transform': 'rotateX(90deg)'
				});
				onDropdownOpen($dropdown);
			}
			this.settings.onDropdownClose = function($dropdown = null) {
				$(this.$dropdown).show().css({
					'position': 'absolute',
					'-webkit-transform': 'rotateX(0deg)',
					'transform': 'rotateX(0deg)',
					'transition': 'transform 0.1s linear, background 0.15s linear, box-shadow 0.15s ease-out'
				});
				onDropdownClose($dropdown);
			}
		}

		// Init selectize
		$(this.$el).selectize({
			onInitialize: function() {
				self.selectize = this;
				self.setValue();
			},
			onChange: value => {
				this.$emit('input', value);
				this.settings.onChange(value);
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
			...this.settings
		})

		// At init
		this.makeOptions(true);
		this.toggleDisabled(this.disabled);

		$(this.$el).find('input').on('input', e => {
			this.inputText = e.target.value;

			// On enter
			if (e.keyCode === 13 && this.settings.createOnEnter && this.focus && this.settings.create) {
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
		if (this.$el.selectize) {
			this.$el.selectize.destroy();
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
		disabled(value) {
			this.toggleDisabled(value)
		}
	},
	methods: {
		toggleDisabled(value) {
			if (value) {
				this.$el.selectize.disable();
			} else {
				this.$el.selectize.enable();
			}
		},
		makeOptions(justLocal = false) {
			var old = this.options
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
				this.options = _new
				if (!justLocal) {
					this.$el.selectize.clearOptions();
					var optionValues = this.options.map(o => o.value)
					Object.keys(this.$el.selectize.options)
						//IE11 fix, Object.values is not supported
						.map(key => this.$el.selectize.options[key]).filter(option => optionValues.every(v => !equal(v, option.value))).forEach(option => this.$el.selectize.removeOption(option.value));
					this.$el.selectize.addOption(this.options);
					this.$el.selectize.refreshOptions(false);
					this.setValue();
				}
			}
		},
		setValue(value) {
			if (!value) value = this.value;
			if (this.settings.forceAdding) {
				this.addOptionsIfNotExists(value);
			}
			this.$el.selectize.setValue(value, true);
			this.log('Value Set: ' + value);
		},
		setOptions(options) {
			// Save selected items before clear options (like backup)
			var items = this.value;

			// Disable onchange event while items readding
			var onchange = this.$el.selectize.onChange;
			this.$el.selectize.onChange = function() {};

			// Add options, clearOptions will remove items too
			this.$el.selectize.clearOptions();
			options.forEach(option => this.$el.selectize.addOption(option));

			// Set items form backup
			this.addItems(items);

			this.$el.selectize.refreshOptions(false)
			this.setValue();

			// Reload onchange event
			this.$el.selectize.onChange = onchange;
		},

		// Add options if array 
		addOptions(options) {

			if (Array.isArray(options)) {
				options.forEach(option => this.$el.selectize.addOption(option));
				return options;
			}
			
			this.addOption(options);
			return options;
		},

		// Add one option
		addOption(option) {
			this.$el.selectize.addOption(option);
			this.$el.selectize.refreshOptions(false);
		},
		setItems(items, force = false) {

			// Disable onchange event while items readding
			var onchange = this.$el.selectize.onChange;
			this.$el.selectize.onChange = function() {};

			// Set items
			this.$el.selectize.clearItems();
			this.addItems(items, force);

			// Reload onchange event
			this.$el.selectize.onChange = onchange;
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
			this.$el.selectize.addItem(items);
		},
		removeItem(item) {
			this.$el.selectize.removeItem(item);
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
			this.options.forEach(function(option) {
				if (option[valueField] === value) {
					found = true;
					return;
				}
			});

			// If option not exists add
			if (found === true) {
				return true;
			}
			var option = {};
			option[valueField] = value;
			option[labelField] = value;
			this.$el.selectize.addOption(option);
			return true;
		},
		addItemAsOption(option) {

			// Find option by valueField
			var valueField = this.settings.valueField || 'value';

			this.$el.selectize.addOptionIfNotExists(option);
			this.$el.selectize.addItem(option[valueField]);
			this.setValue();
		},
		setFocus() {
			this.$el.selectize.focus();
		},
		setBlur() {
			this.$el.selectize.blur();
		},
		log(text) {
			if (this.settings.debug) console.log('Selectize -- ' + text);
		}
	},
	beforeUpdate() {
		this.makeOptions();
	}
} 
</script>
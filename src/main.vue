<template>
  <select>
    <slot/>
  </select>
</template>

<script>
import $ from 'jquery'
import equal from 'deep-equal'

if (!$().selectize) {
  require('selectize')
}

function clean (options) {
  return options
    .map(option => ({
      text: option.text,
      value: option.value
    }))
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
        valueField: 'value',
        slideToggle: true
      })
    },
    disabled: {
      type: Boolean,
      default: false
    }
  },
  data () {
    return {
      selectize: {},
      options: [],
      createdOptions: []
    }
  },
  mounted () {
    var self = this
    if (this.settings.create) {
      var create = this.settings.create
      this.settings.create = function(input,callback) {
        let option = null
        if (create === true) {
          option = {
            text: input,
            value: input
          }
        }
        else {
          option = create(input, callback)
        }
        self.createdOptions.push(option)
        return option
      }
    }

    if (this.settings.slideToggle) {

      var onDropdownOpen = this.settings.onDropdownOpen;
      var onDropdownClose = this.settings.onDropdownClose;

      this.settings.onDropdownOpen = function ($dropdown = null) {
          $(this.$dropdown).hide().slideDown('fast').fadeIn('fast');
          onDropdownOpen($dropdown);
      }

      this.settings.onDropdownClose = function ($dropdown = null) {
          $(this.$dropdown).show().slideUp('fast').fadeOut('fast');
          onDropdownClose($dropdown);
      }
    }

    $(this.$el).selectize({
      onInitialize: function() {
        self.selectize = this;
        self.setValue()
      },
      onChange: value => {
        this.$emit('input', value)
      },
      setOptions: (options) => {
        this.setOptions(options);
      },
      addOptions: (options) => {
        this.addOptions(options);
      },
      addItems: (items) => {
        if (Array.isArray(items)) items.forEach(item => this.$el.selectize.addItem(item));
        else this.$el.selectize.addItem(items);
        this.setValue();
      },
      ...this.settings
    })
    this.makeOptions(true)
    this.toggleDisabled(this.disabled)
  },
  destroyed () {
    if (this.$el.selectize) {
      this.$el.selectize.destroy()
    }
  },
  watch: {
    value (value, old) {
      if (!equal(value, old)) {
        this.setValue()
      }
    },
    disabled (value) {
      this.toggleDisabled(value)
    }
  },
  methods: {
    toggleDisabled (value) {
      if (value) {
        this.$el.selectize.disable()
      }
      else {
        this.$el.selectize.enable()
      }
    },
    makeOptions (justLocal = false) {
      var old = this.options
      let _new = []
      var nodes = this.$slots.default
      if (this.settings.options === undefined && nodes) {
        _new = nodes
          .filter(node => node.tag && node.tag.toLowerCase() === 'option')
          .map(node => {
            return {
              text: node.children ? node.children[0].text.trim() : null,
              value: node.data.domProps ? node.data.domProps.value : node.data.attrs.value
            }
          })
          .concat(this.createdOptions)
      }
      if (!equal(clean(old), clean(_new))) {
        this.options = _new
        if (!justLocal) {
          this.$el.selectize.clearOptions();
          var optionValues = this.options.map(o => o.value)
          Object.keys(this.$el.selectize.options)
            //IE11 fix, Object.values is not supported
            .map(key => this.$el.selectize.options[key])
            .filter(option => optionValues.every(v => !equal(v, option.value)))
            .forEach(option => this.$el.selectize.removeOption(option.value))
          this.$el.selectize.addOption(this.options)
          this.$el.selectize.refreshOptions(false)
          this.setValue()
        }
      }
    },
    setValue () {
      console.log('selectize value', this.value);
      if (this.settings.forceAdding) {
        this.addOptionsIfNotExists(this.value);
      }
      this.$el.selectize.setValue(this.value, true)
    },
    setOptions (options) {
      var items = this.value;
      var onchange = this.$el.selectize.onChange;
      this.$el.selectize.onChange = function() {};
      this.$el.selectize.clearOptions();
      options.forEach(option => this.$el.selectize.addOption(option));
      if (Array.isArray(items)) {
        items.forEach(item => this.$el.selectize.addItem(item));
      } else {
        this.$el.selectize.addItem(items);
      }
      this.$el.selectize.refreshOptions(false)
      this.setValue();
      this.$el.selectize.onChange = onchange;
    },
    addOptions (options) {
      options.forEach(option => this.$el.selectize.addOption(option));
      this.$el.selectize.refreshOptions(false)
      this.setValue()
    },
    addOption (option) {

    },
    addItems (items, force = false) {

      if (Array.isArray(items)) {
        items.forEach(item => {
          if (force) this.addOptionIfNotExists(item)
          this.$el.selectize.addItem(item)
        });
      } else {
        if (force) this.addOptionIfNotExists(item)
        this.$el.selectize.addItem(items)
      }

      this.setValue()
    },
    removeItem (item) {
      this.$el.selectize.removeItem(item)
      this.setValue()
    },
    addOptionsIfNotExists (values) {
      var vm = this;
      values.forEach(function(value) {
        vm.addOptionIfNotExists(value);
      });
    },
    addOptionIfNotExists (value) {
      var found = false;
      var valueField = this.settings.valueField || 'value';
      var labelField = this.settings.labelField || 'text';

      // Find value
      this.options.forEach(function(option) {
        if (option[valueField] === value) {
          found = true;
          return;
        }
      });

      if (found === false) {
        var option = {};
        option[valueField] = value;
        option[labelField] = value;
        this.$el.selectize.addOption(option)
        this.$el.selectize.refreshOptions(false)
        
      }
    },
    addItemForce (option) {
      if (!this.options.find(o => o === option)) {
        var valueField = this.settings.valueField || 'value';
        this.$el.selectize.addOption(option);
        this.$el.selectize.refreshOptions(false);
        this.$el.selectize.addItem(option[valueField]);
        this.setValue();
      }
    }
  },
  beforeUpdate () {
    this.makeOptions();
  }
}
</script>
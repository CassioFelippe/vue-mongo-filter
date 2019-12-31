<template>
  <div>
    <b-input-group :size="mobile('sm', 'lg')">
      <template v-if="mobile(true, false)" v-slot:prepend>
        <b-input-group-text>{{App.t('filtrar')}}</b-input-group-text>
      </template>

      <b-form-select id="filter-attribute" :options="fields" v-model="filter.key" text-field="label" value-field="key" :searchable="false">
        <template v-slot:first>
          <option :value="''" />
        </template>
      </b-form-select>

      <b-form-select id="filter-operator" :options="operators" v-model="filter.operator" text-field="label" :searchable="false" />

      <b-form-select
        v-if="selected.values"
        id="filter-value"
        :options="selected.values"
        v-model="filter.value"
        :searchable="false"
        title="Selecione o valor que deseja filtrar"
      >
        <template v-slot:first>
          <option :value="{}" />
        </template>
      </b-form-select>

      <the-mask
        v-else-if="filter.key && selected.mask"
        id="filter-value"
        pattern="\d*"
        @keypress.enter.native="addFilter()"
        v-model="filter.value"
        :mask="selected.mask"
        :placeholder="selected.mask ? selected.mask.replace(/#/g, '_') : ''"
        class="form-control"
      />
      
      <b-form-input
        v-else
        id="filter-value"
        :pattern="selected.pattern"
        :type="selected.type"
        @keypress.enter="addFilter()"
        v-model="filter.value"
        title="Digite o valor que deseja filtrar"
      />
      
      <template v-slot:append>
        <b-button id="filter-search" squared @click="addFilter()" variant="outline-primary" :title="readyToFilter ? 'Filtrar' : 'Atualizar'">
          <font-awesome-icon :icon="readyToFilter ? 'filter' : 'sync'"/>
        </b-button>
        <b-button id="filter-clear" squared @click="clearFilters()" variant="outline-danger" title="Limpar Filtro">
          <font-awesome-icon icon="trash"/>
        </b-button>
      </template>
    </b-input-group>
    
    <b-col>&nbsp;</b-col>

    <b-input-group :size="mobile('sm', 'lg')" v-for="(values, key, index) in filters" :key="key" class="applied-filter">
      <b-input-group :size="mobile('sm', 'lg')" v-for="(operator, i) in Object.keys(values).filter((k) => k !== '$options')" :key="i">
        <span :class="'form-control applied-key'+index">{{App.t(key)}}</span>
        <span :class="'form-control applied-operator'+index">{{App.t('operators.'+operator)}}</span>
        <span :class="'form-control applied-value'+index">{{values[operator]}}</span>
        <b-button cols="1" @click="removeFilter(key)" :class="'btn-sm remove-applied'+index" squared variant="outline-danger" title="Remover Filtro">
          <font-awesome-icon icon="minus"/>
        </b-button>
      </b-input-group>
    </b-input-group>
  </div>
</template>

<script>
  import * as App from '@/App'

  export default {
    name: 'input-filter',

    props: {
      fields: {
        default: function() {
          return [];
        }
      },
      filters: {},
      model: {},
      prefix: {},
      service: {}
    },

    data() {
      return {
        filter: {
          type: 'text',
          operator: '$eq',
          key: '',
          value: null
        },
        operators: this.updateOperators(),
        App: App,
        mobile: function(primary, secondary) {
          let userAgent = navigator.userAgent,
          isMobile = userAgent.match(/Android/i)
            || userAgent.match(/webOS/i)
            || userAgent.match(/iPhone/i)
            || userAgent.match(/iPad/i)
            || userAgent.match(/iPod/i)
            || userAgent.match(/BlackBerry/i)
            || userAgent.match(/Windows Phone/i);
          return isMobile ? secondary : primary;
        }
      };
    },
    
    computed: {
      selected: function() {
        let filtered = this.fields.filter(f => f.key === this.filter.key);
        return this.fields && filtered[0] ? filtered[0] : {};
      },

      readyToFilter: function() {
        return this.filter.key && this.filter.value;
      }
    },

    watch: {
      'filter.key': function(value) {
        let field = this.fields.filter(f => f.key === value)[0], type = field ? field.type : null;
        this.operators = this.updateOperators(type);
      }
    },

    methods: {
      request(filters) {
        // add filters to store to keep them after changing routes
        this.$store.filters = filters;

        // clean fields of filter selection
        this.filter = {type: 'text', operator: '$eq'};

        this.service.post(`${this.prefix}/filter`, filters).then(response => {
          this.$emit('update:model', response.data);
        });
      },

      addCriteria(filters /*applied filters*/, current /*filter to be added*/) {
        let value = {[current.operator]: current.value};

        if (current.operator === '$regex') {
          value = {[current.operator]: current.value, $options: 'i'}
        }

        if (filters[current.key]) {
          Object.assign(filters[current.key], value);
        } else {
          filters[current.key] = value;
        }

        return filters; // TODO check if it's necessary or if it's possible to set the value by reference
      },

      addFilter() {
        if (this.filter.key && this.filter.value) {
          let current = {
            key: this.filter.key,
            operator: this.filter.operator,
            value: this.fields.filter(f => f.key === this.filter.key)[0].type === 'number' ? parseInt(this.filter.value) : this.filter.value
          }

          this.filters = this.addCriteria(this.filters, current);
        }

        this.request(this.filters);
      },

      removeFilter(filter) {
        delete this.filters[filter];
        
        this.request(this.filters);
      },

      clearFilters() {
        this.$emit('update:filters', {});
        
        this.request({});
      },

      updateOperators(type) {
        if (this.filter) {
          this.filter.type = type;
        }

        return type ? list.filter(f => f.types.includes(type)) : list;
      }
    },

    mounted() {}
  };

  const list = [
    {
      label: App.t('operators.$eq'),
      value: '$eq',
      types: ['text', 'number', 'date']
    },
    {
      label: App.t('operators.$ne'),
      value: '$ne',
      types: ['text', 'number', 'date']
    },
    {
      label: App.t('operators.$regex'),
      value: '$regex',
      types: ['text']
    },
    {
      label: App.t('operators.$gt'),
      value: '$gt',
      types: ['number', 'date']
    },
    {
      label: App.t('operators.$gte'),
      value: '$gte',
      types: ['number', 'date']
    },
    {
      label: App.t('operators.$lt'),
      value: '$lt',
      types: ['number', 'date']
    },
    {
      label: App.t('operators.$lte'),
      value: '$lte',
      types: ['number', 'date']
    }
  ];
</script>

<style scoped>
  /* mobile only */
  @media only screen and (max-width: 600px) {
    .input-group-lg > .input-group-prepend > .custom-select {
      border-right: 0px !important;
    }

    .input-group-sm > .input-group-prepend > .custom-select {
      border-right: 0px !important;
    }

    .form-control.sm {
      max-width: 20% !important;
    }

    .input-group > select, .input-group > input, .input-group > span, .input-group-append {
      width: 100%;
    }

    .input-group button {
      width: 50%;
      margin-right: 2px;
    }

    #filter-clear {
      margin-right: -1px;
    }

    #filter-attribute {
      border-radius: 0;
      margin-left: -1px;
    }
  }
</style>
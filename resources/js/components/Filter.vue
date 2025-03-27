<template>
  <FilterContainer>
    <span>{{ filter.name }}</span>
    <template #filter>
      <div class="relative">
        <input type="text" class="hidden">
        <input
            :id="id"
            class="block w-full form-control form-control-sm form-input form-input-bordered text-sm px-1"
            :class="{ 'text-white': (value == null) }"
            type="text"
            :dusk="`${filter.name}-daterange-filter`"
            name="daterangefilter"
            autocomplete="off"
            :value="value"
            :placeholder="placeholder"
            @keydown="handleInput($event)"
            @paste.prevent
        />
        <div
            v-if="value"
            class="absolute top-0 right-0 mt-1 mr-1">
          <button class="bg-transparent" @click="clearFilter">
            <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="w-6 h-6">
              <path stroke-linecap="round" stroke-linejoin="round" d="M6 18 18 6M6 6l12 12" />
            </svg>
          </button>
        </div>
      </div>
    </template>
  </FilterContainer>
</template>

<script>
import debounce from 'lodash/debounce'
import moment from '../../../dist/js/moment.min'
import 'moment/locale/fr'

// Force la configuration de Moment.js en français et assigne à l'instance globale
moment.locale('fr')
window.moment = moment

export default {
  emits: ['change'],

  props: {
    resourceName: {
      type: String,
      required: true,
    },
    filterKey: {
      type: String,
      required: true,
    },
    lens: String,
  },

  data: () => ({
    id: null,
    value: null,
    startDate: null,
    endDate: null,
    currentStartDate: null,
    currentEndDate: null,
    debouncedHandleChange: null,
  }),

  created() {
    this.parseDates()
    this.debouncedHandleChange = debounce(() => this.handleChange(), 500)
    this.setCurrentFilterValue()
  },

  mounted() {
    this.id = 'dateRangeCalendar_' + this.generateId()
    Nova.$on('filter-reset', this.setCurrentFilterValue)
    // Assurez-vous que cette configuration est appliquée après le chargement complet de Nova
    setTimeout(() => {
      this.initDateRange()
    }, 1);
  },

  beforeUnmount() {
    Nova.$off('filter-reset', this.setCurrentFilterValue)
  },

  watch: {
    value() {
      this.debouncedHandleChange()
    },
  },

  methods: {
    setCurrentFilterValue() {
      this.value = this.filter.currentValue || null
    },
    handleChange() {
      this.$store.commit(`${this.resourceName}/updateFilterState`, {
        filterClass: this.filterKey,
        value: (
            (this.currentStartDate && this.currentEndDate) ?
                (this.currentStartDate.format('YYYY-MM-DD') + ' - ' + this.currentEndDate.format('YYYY-MM-DD')) :
                null
        ),
      })
      this.$emit('change')
      this.currentStartDate = null
      this.currentEndDate = null
    },
    handleInput(e) {
      return e.preventDefault()
    },
    initDateRange() {
      const ref = this
      const idSelector = '#' + this.id
      const minDate = ref.filter.minDate
      const maxDate = ref.filter.maxDate

      console.log('moment.locale():', moment.locale()) // Devrait afficher "fr"

      // Forcer la configuration globale du Date Range Picker
      if ($.fn.daterangepicker) {
        $.fn.daterangepicker.defaults.locale = {
          format: 'DD/MM/YYYY',
          separator: ' - ',
          applyLabel: 'Appliquer',
          cancelLabel: 'Annuler',
          fromLabel: 'Du',
          toLabel: 'Au',
          customRangeLabel: 'Personnalisé',
          weekLabel: 'S',
          daysOfWeek: ['Dim', 'Lun', 'Mar', 'Mer', 'Jeu', 'Ven', 'Sam'],
          monthNames: [
            'Janvier', 'Février', 'Mars', 'Avril', 'Mai', 'Juin',
            'Juillet', 'Août', 'Septembre', 'Octobre', 'Novembre', 'Décembre'
          ],
          firstDay: 1
        }
      }

      $(idSelector).daterangepicker({
        startDate: ref.startDate,
        endDate: ref.endDate,
        minDate: (minDate ? moment(minDate) : null),
        maxDate: (maxDate ? moment(maxDate) : null),
        ranges: ref.parseRanges(),
        locale: {
          format: 'DD/MM/YYYY',
          separator: ' - ',
          applyLabel: 'Appliquer',
          cancelLabel: 'Annuler',
          fromLabel: 'Du',
          toLabel: 'Au',
          daysOfWeek: ['Dim', 'Lun', 'Mar', 'Mer', 'Jeu', 'Ven', 'Sam'],
          monthNames: [
            'Janvier', 'Février', 'Mars', 'Avril', 'Mai', 'Juin',
            'Juillet', 'Août', 'Septembre', 'Octobre', 'Novembre', 'Décembre'
          ],
          firstDay: 1
        },
      }, function(start, end, label) {
        if (start && end) {
          ref.currentStartDate = start
          ref.currentEndDate = end
        }
      })
          .on('apply.daterangepicker', function(ev, picker) {
            if (ref.currentStartDate && ref.currentEndDate) {
              ref.value = ref.currentStartDate.format('DD/MM/YYYY') + ' - ' + ref.currentEndDate.format('DD/MM/YYYY')
            }
          })
    },
    clearFilter() {
      this.value = null
    },
    generateId() {
      return Math.random().toString(36).substring(2) + (new Date()).getTime().toString(36)
    },
    parseDates() {
      const dateRange = this.filter.currentValue
      let startDate = moment()
      let endDate = moment()

      if (dateRange) {
        const parsedDateRange = dateRange.split(' to ')
        if (parsedDateRange.length === 2) {
          try {
            startDate = moment(parsedDateRange[0], "YYYY-MM-DD")
            endDate = moment(parsedDateRange[1], "YYYY-MM-DD")
          } catch(e){}
        }
      }
      // Utiliser le format français partout
      this.startDate = startDate.format('DD/MM/YYYY')
      this.endDate = endDate.format('DD/MM/YYYY')
      this.currentStartDate = startDate
      this.currentEndDate = endDate
    },
    parseRanges() {
      const ranges = this.filter.options
      let parsedRanges = {}
      for (let i = 0; i < ranges.length; i++) {
        parsedRanges[ranges[i]['label']] = [moment(ranges[i][0]), moment(ranges[i][1])]
      }
      return parsedRanges
    }
  },

  computed: {
    filter() {
      return this.$store.getters[`${this.resourceName}/getFilter`](this.filterKey)
    },
  },
}
</script>

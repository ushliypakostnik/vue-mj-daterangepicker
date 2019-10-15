<template lang="pug">
  .mj-daterange-picker(:style="cssProps")
    .panels-choices(v-if="availablePanels.length > 1")
      .panel-button(
        v-for="panel in availablePanels"
        :class="{'is-current': panel === currentPanel}"
        @click="currentPanel = panel"
        ) {{ $legends[locale].panels[panel] }}

    .preset-ranges(v-if="isPresetPicker && presets.length > 1")
      .preset(v-for="entry in availablePresets")
        input(type="radio" v-model="preset" :id="entry" :value="entry")
        label(:for="entry")
          span.check
          span {{ $legends[locale].presets[entry] }}

    .mj-calendar(:class="weekSelector ? 'mj-calendar-week' : 'mj-calendar-days'" v-if="isDaysPicker")
      .calendar-header
        .calendar-previous-month.calendar-arrow.calendar-arrow-previous(
          :aria-label="$legends[locale].previousMonth"
          @click="changeMonth(1)"
        )
          svgicon(icon="arrow-left" width="7.4" height="12")
        .calendar-month-name {{ currentMonthName }}
        .calendar-previous-month.calendar-arrow.calendar-arrow-next(
          :aria-label="$legends[locale].nextMonth"
          @click="changeMonth(-1)")
          svgicon(icon="arrow-right" width="7.4" height="12")
      .calendar-days-name
        .day(v-for="day in firstWeek")
          span {{ day.name }}
      .calendar-days
        .day(
          v-for="day in monthDays"
          :key="day.date | date('DDMMYYYY')"
          :class="dayClasses(day)"
          @click="selectDay(day.date)"
          @mouseover="hoverizeDay(day.date)"
          @mouseleave="hoverRange = []"
        )
          span {{ day.date | date('D') }}

    .mj-calendar(v-if="isMonthsPicker")
      .calendar-header
        .calendar-previous-month.calendar-arrow.calendar-arrow-previous(
          :aria-label="$legends[locale].previousYear"
          @click="changeYear(1)"
        )
          svgicon(icon="arrow-left" width="7.4" height="12")
        .calendar-month-name {{ currentYearName }}
        .calendar-previous-month.calendar-arrow.calendar-arrow-next(
          :aria-label="$legends[locale].nextYear"
          @click="changeYear(-1)")
          svgicon(icon="arrow-right" width="7.4" height="12")

      .calendar-months(v-if="isMonthsPanel")
        .month(
          v-for="month in yearMonths"
          :key="month.date | date('DDMMYYYY')"
          :class="monthClasses(month)"
          @click="selectMonth(month)"
          @mouseover="hoverizeMonth(month.date)"
          @mouseleave="hoverRange = []"
        )
          span {{ month.displayDate }}

      .calendar-quarters(v-if="isQuartersPanel")
        .quarter(
          v-for="(quarter, index) in yearQuarters"
          :key="index"
          :class="quarterClasses(quarter)"
          @click="selectQuarter(quarter)"
          @mouseover="hoverizeQuarter(quarter.range.start, quarter.range.end)"
          @mouseleave="hoverRange = []"
          )
          .legend {{ $legends[locale].quarter }} {{ ++index }}
          .months
            .month(v-for="month in quarter.months")
              span {{ month.displayDate }}

    .mj-calendar(v-if="isYearPicker")
      .calendar-years
        .year(
          v-for="(year, index) in years"
          :key="index"
          :class="yearClasses(year)"
          @click="selectYear(year)"
          @mouseover="hoverizeYear(year)"
          @mouseleave="hoverRange = []"
        )
          span {{ year.displayDate }}

    .mj-daterange-picker-controls(v-if="showControls")
      .mj-daterange-picker-button.mj-daterange-picker-reset(
        @click="reset"
      )
        | {{ resetLegend }}
      .mj-daterange-picker-button.mj-daterange-picker-submit(
        @click="update"
        :class="{'is-disabled': !(values.from && values.to) }"
      )
        | {{ submitLegend }}
</template>

<script lang="ts">
  import { Component, Prop, Vue, Watch } from 'vue-property-decorator'
  import SvgIcon from 'vue-svgicon'
  import './../assets/icons'
  import { dateFilter } from 'vue-date-fns'
  import {
    addDays,
    addMonths,
    addWeeks,
    addYears,
    differenceInDays,
    endOfDay,
    endOfMonth,
    endOfWeek,
    endOfYear,
    format,
    isAfter,
    isBefore,
    isSameDay,
    isSameMonth,
    isValid,
    isWithinRange,
    parse,
    startOfDay,
    startOfMonth,
    startOfWeek,
    startOfYear,
    subDays,
    subMonths,
    subWeeks,
    subYears
  } from 'date-fns'

  import dictionnaries from '../translations/index.js'

  Vue.prototype.$legends = dictionnaries

  const locales = {
    en: require('date-fns/locale/en'),
    fr: require('date-fns/locale/fr'),
    de: require('date-fns/locale/de'),
    es: require('date-fns/locale/es'),
    ru: require('date-fns/locale/ru'),
  }

  const RANGE_PANELS = [ 'days', 'weeks', 'months', 'quarters', 'years']
  const SINGLE_PANELS = [ 'day', 'week', 'month', 'quarter', 'year']

  Vue.use(SvgIcon, {
    tagName: 'svgicon'
  })

  @Component({
    filters: {
      date: dateFilter
    }
  })
  export default class extends Vue {
    $legends: any
    currentPanel = null

    current = null
    weekSelector = false
    isWeeksRangeOpen = false
    isMonthsRangeOpen = false
    isQuartersRangeOpen = false
    isYearsRangeOpen = false
    monthDays = []
    now = new Date().toISOString()
    start = null
    end = null
    values = {
      from: null,
      to: null
    }
    hoverRange = []
    preset = 'custom'

    @Prop({
      type: Boolean,
      default: true
    }) range

    @Prop({
      type: String,
      default: 'en'
    }) locale

    @Prop({
      type: String,
      default: null
    }) from

    @Prop({
      type: String,
      default: null
    }) to

    @Prop({
      type: String,
      default: null
    }) begin

    @Prop({
      type: Boolean,
      default: true
    }) past

    @Prop({
      type: Boolean,
      default: true
    }) future

    @Prop({
      type: Boolean,
      default: true
    }) showControls

    @Prop({
      type: String,
      default: null
    }) panel

    @Prop({
      type: Number,
      default: 2
    }) yearsCount

    @Prop({
      type: Number,
      default: null
    }) yearsPast

    @Prop({
      type: Number,
      default: null
    }) yearsFuture

    @Prop({
      type: String,
      default: null
    }) allowFrom

    @Prop({
      type: String,
      default: null
    }) allowTo

    @Prop({
      type: Object,
      default: () => {
        return {
          primary: '#ebf5fe',
          secondary: '#b1b1b1',
          ternary: '#b1b1b1',
          border: '#dddddd',
          light: '#ffffff',
          dark: '#373a3c',
          hovers: {
            day: '#def0d7',
            range: '#def0d7'
          }
        }
      }
    }) theme

    @Prop({
      type: String,
      default: 'auto'
    }) width

    @Prop({
      type: String,
      default: null
    }) resetTitle

    @Prop({
      type: String,
      default: null
    }) submitTitle

    @Prop({
      type: Array,
      default: () => ['today', 'yesterday', 'last7days', 'last30days', 'last90days', 'last365days', 'forever', 'custom']
    }) presets

    @Watch('currentPanel', { immediate: true })
    switchMode(panel) {
      this.weekSelector = panel === 'week' || panel === 'weeks' ? true : false
      this.updateCalendar()
    }

    @Watch('values', { deep: true })
    emitValuesWithSelect(values) {
      if (values.from && values.to) {
        this.$emit('select', {
          to: format(endOfDay(this.values.to), 'YYYY-MM-DDTHH:mm:ss.SSSZ'),
          from: format(startOfDay(this.values.from), 'YYYY-MM-DDTHH:mm:ss.SSSZ'),
          panel: this.currentPanel
        })
      }
    }

    @Watch('preset')
    affectPreset(preset) {
      this.current = this.now
      this.updateCalendar()

      switch (preset) {
        case 'custom':
          this.values = { from: null, to: null }
          break
        case 'today':
          this.values = { from: startOfDay(this.now), to: this.now }
          break
        case 'yesterday':
          this.values = { from: startOfDay(subDays(this.now, 1)), to: endOfDay(subDays(this.now, 1)) }
          break
        case 'tomorrow':
          this.values = { from: startOfDay(addDays(this.now, 1)), to: endOfDay(addDays(this.now, 1)) }
          break
        case 'last7days':
          this.values = { from: startOfDay(subWeeks(this.now, 1)), to: this.now }
          break
        case 'next7days':
          this.values = { to: startOfDay(addWeeks(this.now, 1)), from: this.now }
          break
        case 'last30days':
          this.values = { from: startOfDay(subMonths(this.now, 1)), to: this.now }
          break
        case 'next30days':
          this.values = { to: startOfDay(addMonths(this.now, 1)), from: this.now }
          break
        case 'last90days':
          this.values = { from: startOfDay(subMonths(this.now, 3)), to: this.now }
          break
        case 'next90days':
          this.values = { to: startOfDay(addMonths(this.now, 3)), from: this.now }
          break
        case 'last365days':
          this.values = { from: startOfDay(subYears(this.now, 1)), to: this.now }
          break
        case 'next365days':
          this.values = { to: startOfDay(addYears(this.now, 1)), from: this.now }
          break
        case 'forever':
          this.values = { from: this.begin, to: this.now }
          break
      }
    }

    get availablePanels() {
      if (this.range) {
        return RANGE_PANELS
      }
      return SINGLE_PANELS
    }

    get availablePresets() {
      const index = this.presets.indexOf('forever')
      if (!this.begin && index > -1 ) {
        this.presets.splice(index, 1)
      }
      return this.presets
    }

    get resetLegend() {
      return this.resetTitle ? this.resetTitle : this.$legends[this.locale].reset
    }

    get submitLegend() {
      return this.submitTitle ? this.submitTitle : this.$legends[this.locale].submit
    }

    get firstWeek() {
      const days = this.monthDays.slice(0, 7)
      const week = []
      for (const day of days) {
        week.push({
          name: format(day.date, 'dd', { locale: locales[this.locale] })
        })
      }
      return week
    }

    get cssProps() {
      return {
        '--default-width': this.width,
        '--primary-color': this.theme.primary,
        '--hover-day-color': this.theme.hovers.day,
        '--hover-range-color': this.theme.hovers.range,
        '--secondary-color': this.theme.secondary,
        '--ternary-color': this.theme.ternary,
        '--normal-color': this.theme.light,
        '--contrast-color': this.theme.dark,
        '--border-color': this.theme.border
      }
    }

    get yearMonths() {
      const months = []
      let month = startOfYear(this.current)

      while (months.length !== 12) {
        const isMonthAllowed = this.isRangeAllowed([startOfMonth(month), endOfMonth(month)])
        const str1 = format(month, 'MMMM', { locale: locales[this.locale] })
        const str2 = str1[0].toUpperCase() + str1.slice(1)

        months.push({
          date: month,
          selectable: isMonthAllowed,
          displayDate: str2
        })
        month = addMonths(month, 1)
      }
      return months
    }

    get yearQuarters() {
      const quarters = []
      for (const [index, month] of this.yearMonths.entries()) {
        if (index % 3 === 0) {
          const isQuarterAllowed = this.isRangeAllowed([startOfMonth(this.yearMonths[index].date), endOfMonth(this.yearMonths[index + 2].date)])

          quarters.push({
            months: [
              this.yearMonths[index],
              this.yearMonths[index + 1],
              this.yearMonths[index + 2]
            ],
            selectable: isQuarterAllowed,
            range: {
              start: startOfDay(startOfMonth(this.yearMonths[index].date)),
              end: endOfDay(endOfMonth(this.yearMonths[index + 2].date))
            }
          })
        }
      }
      return quarters
    }

    get years() {
      const years = []
      const futureCount = this.yearsFuture ? this.yearsFuture : this.yearsCount
      const pastCount = this.yearsPast ? this.yearsPast : this.yearsCount
      let i: number = +this.future * futureCount + +this.past * pastCount + 1
      let start = this.future ? addYears(this.now, futureCount) : this.now

      while (i !== 0) {
        const isYearAllowed = this.isRangeAllowed([startOfYear(start), endOfYear(start)])

        years.push({
          date: start,
          selectable: isYearAllowed,
          displayDate: format(start, 'YYYY', { locale: locales[this.locale] })
        })
        start = subYears(start, 1)
        i = i - 1
      }
      return years.reverse()
    }

    get currentMonthName() {
      return format(this.current, 'MMMM YYYY', { locale: locales[this.locale] })
    }

    get currentYearName() {
      return format(this.current, 'YYYY', { locale: locales[this.locale] })
    }

    get isPresetPicker(): boolean {
      return this.currentPanel === 'days'
    }

    get isDaysPicker(): boolean {
      return this.currentPanel === 'days' || this.currentPanel === 'weeks' || this.currentPanel === 'week' || this.currentPanel === 'day'
    }

    get isMonthsPicker(): boolean {
      return this.currentPanel === 'months' || this.currentPanel === 'month' || this.currentPanel === 'quarter' || this.currentPanel === 'quarters'
    }

    get isYearPicker(): boolean {
      return this.currentPanel === 'year' || this.currentPanel === 'years'
    }

    get isMonthsPanel(): boolean {
      return this.currentPanel === 'months' || this.currentPanel === 'month'
    }

    get isQuartersPanel(): boolean {
      return this.currentPanel === 'quarter' || this.currentPanel === 'quarters'
    }

    created() {
      // Parse Inputs
      Object.keys(this.values).forEach((value) => {
        this.values[value] = isValid(parse(this[value])) ? this[value] : null
      })

      // Todo ? If from or to is null, or from is after to, both are null

      // Display current month or "to" month
      this.current = this.values.to ? this.values.to : this.now

      // Update Calendar
      this.updateCalendar()

      this.start = startOfYear(this.years[0].date)
      this.end = endOfYear(this.years[this.years.length - 1].date)

      // Set current panel
      this.currentPanel = this.panel || this.availablePanels[0]
    }

    reset() {
      this.values = {
        to: null,
        from: null
      }
      this.preset = null
      this.$emit('reset', { to: null, from: null })
    }

    update() {
      if (!this.values.from || !this.values.to) {
        return
      }
      this.$emit('update', {
        to: format(endOfDay(this.values.to), 'YYYY-MM-DDTHH:mm:ss.SSSZ'),
        from: format(startOfDay(this.values.from), 'YYYY-MM-DDTHH:mm:ss.SSSZ'),
        panel: this.currentPanel
      })
    }

    changeMonth(diff: number) {
      this.current = subMonths(this.current, diff)
      this.updateCalendar()
    }

    changeYear(diff: number) {
      this.current = subYears(this.current, diff)
      this.updateCalendar()
    }

    selectDay(date) {
      let range

      // Select weeks
      if (this.weekSelector) {
        if (!this.range) {
          // Select week single
          range = this.getAllowedDatesOfRange([startOfWeek(date, { weekStartsOn: 1 }), endOfWeek(date, { weekStartsOn: 1 })])
          this.values.from = range[0]
          this.values.to = range[range.length - 1]
          return
        } else {
          // Select weeks range
          this.isMonthsRangeOpen = false
          this.isQuartersRangeOpen = false
          this.isYearsRangeOpen = false
          if (!this.values.from && !this.values.to) {
            range = this.getAllowedDatesOfRange([startOfWeek(date, { weekStartsOn: 1 }), endOfWeek(date, { weekStartsOn: 1 })])
            this.isWeeksRangeOpen = true
          } else {
            if (isWithinRange(date, this.values.from, this.values.from)) {
              // On selected weeks
              range = this.getAllowedDatesOfRange([startOfWeek(date, { weekStartsOn: 1 }), endOfWeek(date, { weekStartsOn: 1 })])
            } else if (isBefore(date, this.values.from)) {
              // Before selected weeks
              range = this.getAllowedDatesOfRange([startOfWeek(date, { weekStartsOn: 1 }), endOfWeek(date, { weekStartsOn: 1 })])
              this.isWeeksRangeOpen = true
            } else {
              // After selected weeks
              if (this.isWeeksRangeOpen) {
                range = this.getAllowedDatesOfRange([this.values.from, endOfWeek(date, { weekStartsOn: 1 })])
              } else {
                range = this.getAllowedDatesOfRange([startOfWeek(date, { weekStartsOn: 1 }), endOfWeek(date, { weekStartsOn: 1 })])
              }
              this.isWeeksRangeOpen = !this.isWeeksRangeOpen
            }
          }
          this.values.from = range[0]
          this.values.to = range[range.length - 1]
          this.preset = 'custom'
          return
        }
      }

      // Select days
      if (!this.range) {
        // Select day single
        this.values.from = startOfDay(date)
        this.values.to = startOfDay(date)
        return
      } else {
        // Select days range
        this.isWeeksRangeOpen = false
        this.isMonthsRangeOpen = false
        this.isQuartersRangeOpen = false
        this.isYearsRangeOpen = false
        if ((this.values.from && this.values.to) || (!this.values.from && !this.values.to)) {
          this.values.from = date
          this.values.to = null
        } else if (this.values.from && !this.values.to) {
          if (isBefore(date, this.values.from)) {
            this.values.from = date
          } else {
            this.values.to = date
            this.hoverRange = []
          }
        }
        this.preset = 'custom'
      }
    }

    selectMonth(month) {
      let range

      if (!this.range) {
        // Select month single
        range = this.getAllowedDatesOfRange([startOfMonth(month.date), endOfMonth(month.date)])
        this.values.from = range[0]
        this.values.to = range[range.length - 1]
        this.current = this.values.to
        return
      } else {
        // Select months range
        this.isWeeksRangeOpen = false
        this.isQuartersRangeOpen = false
        this.isYearsRangeOpen = false
        if ((!this.values.from && !this.values.to) || (this.values.from && !this.values.to)) {
          range = this.getAllowedDatesOfRange([startOfMonth(month.date), endOfMonth(month.date)])
          this.current = this.values.to
          this.isMonthsRangeOpen = true
        } else {
          if (isWithinRange(month.date, this.values.from, this.values.to)) {
            // On selected months
            if (!this.isMonthsRangeOpen) {
              range = this.getAllowedDatesOfRange([startOfMonth(month.date), endOfMonth(month.date)])
            } else {
              range = this.getAllowedDatesOfRange([this.values.from, endOfMonth(month.date)])
            }
            this.isMonthsRangeOpen = !this.isMonthsRangeOpen
          } else if (isBefore(month.date, this.values.from)) {
            // Before selected months
            range = this.getAllowedDatesOfRange([startOfMonth(month.date), endOfMonth(month.date)])
            this.isMonthsRangeOpen = true
          } else {
            // After selected months
            if (this.isMonthsRangeOpen) {
              range = this.getAllowedDatesOfRange([this.values.from, endOfMonth(month.date)])
            } else {
              range = this.getAllowedDatesOfRange([startOfMonth(month.date), endOfMonth(month.date)])
            }
            this.isMonthsRangeOpen = !this.isMonthsRangeOpen
          }
        }
        this.values.from = range[0]
        this.values.to = range[range.length - 1]
        this.current = this.values.to
        this.preset = 'custom'
      }
    }

    selectQuarter(quarter) {
      let range

      if (!this.range) {
        // Select quarter single
        range = this.getAllowedDatesOfRange([startOfDay(startOfMonth(quarter.range.start)), endOfMonth(quarter.range.end)])
        this.values.from = range[0]
        this.values.to = range[range.length - 1]
        this.current = this.values.to
        return
      } else {
        // Select quarters range
        this.isWeeksRangeOpen = false
        this.isMonthsRangeOpen = false
        this.isYearsRangeOpen = false
        if ((!this.values.from && !this.values.to) || (this.values.from && !this.values.to)) {
          range = this.getAllowedDatesOfRange([startOfDay(startOfMonth(quarter.range.start)), endOfMonth(quarter.range.end)])
          this.isQuartersRangeOpen = true
        } else {
          if (isWithinRange(quarter.range.start, this.values.from, this.values.to)) {
            // On selected quarters
            if (!this.isQuartersRangeOpen) {
              range = this.getAllowedDatesOfRange([startOfDay(startOfMonth(quarter.range.start)), endOfMonth(quarter.range.end)])
            } else {
              range = this.getAllowedDatesOfRange([this.values.from, endOfMonth(quarter.range.end)])
            }
            this.isQuartersRangeOpen = !this.isQuartersRangeOpen
          } else if (isBefore(quarter.range.start, this.values.from)) {
            // Before selected quarters
            range = this.getAllowedDatesOfRange([startOfDay(startOfMonth(quarter.range.start)), endOfMonth(quarter.range.end)])
            this.isQuartersRangeOpen = true
          } else {
            // After selected quarters
            if (this.isQuartersRangeOpen) {
              range = this.getAllowedDatesOfRange([this.values.from, endOfMonth(quarter.range.end)])
            } else {
              range = this.getAllowedDatesOfRange([startOfDay(startOfMonth(quarter.range.start)), endOfMonth(quarter.range.end)])
            }
            this.isQuartersRangeOpen = !this.isQuartersRangeOpen
          }
        }
        this.values.from = range[0]
        this.values.to = range[range.length - 1]
        this.current = this.values.to
        this.preset = 'custom'
      }
    }

    selectYear(year) {
      let range

      if (!this.range) {
        // Select year single
        range = this.getAllowedDatesOfRange([startOfYear(year.date), endOfYear(year.date)])
        this.values.from = range[0]
        this.values.to = range[range.length - 1]
        this.current = this.values.to
        return
      } else {
        // Select years range
        this.isWeeksRangeOpen = false
        this.isMonthsRangeOpen = false
        this.isQuartersRangeOpen = false
        if ((!this.values.from && !this.values.to) || (this.values.from && !this.values.to)) {
          range = this.getAllowedDatesOfRange([startOfYear(year.date), endOfYear(year.date)])
          this.isYearsRangeOpen = true
        } else {
          if (isWithinRange(year.date, this.values.from, this.values.to)) {
            // On selected years
            if (!this.isYearsRangeOpen) {
              range = this.getAllowedDatesOfRange([startOfYear(year.date), endOfYear(year.date)])
            } else {
              range = this.getAllowedDatesOfRange([this.values.from, endOfYear(year.date)])
            }
            this.isYearsRangeOpen = !this.isYearsRangeOpen
          } else if (isBefore(startOfYear(year.date), this.values.from)) {
            // Before selected years
            range = this.getAllowedDatesOfRange([startOfYear(year.date), endOfYear(year.date)])
            this.isYearsRangeOpen = true
          } else {
            // After selected years
            if (this.isYearsRangeOpen) {
              range = this.getAllowedDatesOfRange([this.values.from, endOfYear(year.date)])
            } else {
              range = this.getAllowedDatesOfRange([startOfYear(year.date), endOfYear(year.date)])
            }
            this.isYearsRangeOpen = !this.isYearsRangeOpen
          }
        }
        this.values.from = range[0]
        this.values.to = range[range.length - 1]
        this.current = this.values.to
        this.preset = 'custom'
      }
    }

    hoverizeDay(date) {
      if (!this.weekSelector && (!(this.values.from && !this.values.to) || (isBefore(date, this.values.from)))) {
        this.hoverRange = []
        return
      }
      if (this.weekSelector) {
        let availableRange = []

        if (this.range) {
          if ((!this.values.from && !this.values.to) || (this.values.from && !this.values.to)) {
            availableRange = this.getAllowedDatesOfRange([startOfWeek(date, { weekStartsOn: 1 }), endOfWeek(date, { weekStartsOn: 1 })])
          } else {
            if (isBefore(date, this.values.from)) {
              availableRange = this.getAllowedDatesOfRange([startOfWeek(date, { weekStartsOn: 1 }), endOfWeek(date, { weekStartsOn: 1 })])
            } else {
              if (this.isWeeksRangeOpen) {
                availableRange = this.getAllowedDatesOfRange([startOfWeek(this.values.from, { weekStartsOn: 1 }), endOfWeek(date, { weekStartsOn: 1 })])
              } else {
                availableRange = this.getAllowedDatesOfRange([startOfWeek(date, { weekStartsOn: 1 }), endOfWeek(date, { weekStartsOn: 1 })])
              }
            }
          }
        } else {
          availableRange = this.getAllowedDatesOfRange([startOfWeek(date, { weekStartsOn: 1 }), endOfWeek(date, { weekStartsOn: 1 })])
        }
        this.hoverRange = [availableRange[0], availableRange[availableRange.length - 1]]
      } else {
        this.hoverRange = [this.values.from, date]
      }
    }

    hoverizeMonth(date) {
      if (this.range && !isBefore(date, this.values.from) && this.isMonthsRangeOpen) {
        this.hoverRange = [startOfMonth(this.values.from), endOfMonth(date)]
        return
      } else {
        this.hoverRange = []
      }
    }

    hoverizeQuarter(start, end) {
      if (this.range && !isBefore(start, this.values.from) && this.isQuartersRangeOpen) {
        this.hoverRange = [this.values.from, end]
        return
      } else {
        this.hoverRange = []
      }
    }

    hoverizeYear(year) {
      if (this.range && !isBefore(startOfYear(year.date), this.values.from) && this.isYearsRangeOpen) {
        this.hoverRange = [this.values.from, endOfYear(year.date)]
        return
      } else {
        this.hoverRange = []
      }
    }

    updateCalendar() {
      const days = []

      const lastDayOfMonth = endOfMonth(this.current)
      const firstDayOfMonth = startOfMonth(this.current)
      const nbDaysLastMonth = (+format(firstDayOfMonth, 'd') - 1) % 7

      let day = subDays(firstDayOfMonth, nbDaysLastMonth)

      while (isBefore(day, lastDayOfMonth) || days.length % 7 > 0) {
        const isAllowedByFutureAndPast =
        this.future && isAfter(day, this.now)
          ? true
          : false || (this.past && isBefore(day, this.now))
          ? true
          : false || isSameDay(day, this.now)
        const isAllowedByAllowedProps = this.isDateAllowed(day)
        days.push({
          date: day,
          selectable: isAllowedByFutureAndPast && isAllowedByAllowedProps,
          currentMonth: isSameMonth(this.current, day)
        })
        day = addDays(day, 1)
      }
      this.monthDays = days
    }

    dayClasses(day) {
      const classes = []

      if (day.currentMonth) {
        classes.push('is-current-month')
      }
      if (this.values.from && this.values.to && isWithinRange(day.date, this.values.from, this.values.to) ) {
        classes.push('is-selected')
      }
      if (!day.selectable) {
        classes.push('is-disabled')
      }
      if (isSameDay(day.date, this.now)) {
        classes.push('is-today')
      }
      if (
        (!this.values.to && isSameDay(day.date, this.values.from)) ||
        (this.values.to && !this.values.from &&
        isSameDay(day.date, this.values.from) && isSameDay(day.date, this.values.to)) ||
        (this.values.to && this.values.from && isSameDay(day.date, this.values.from))) {
        classes.push('is-first-range')
        classes.push('is-edge-range')
      }
      if (this.values.to && isSameDay(day.date, this.values.to)) {
        classes.push('is-last-range')
        classes.push('is-edge-range')
      }
      if (this.hoverRange.length === 2 && isWithinRange(day.date, this.hoverRange[0], this.hoverRange[1])) {
        classes.push('is-in-range')
      }
      return classes
    }

    monthClasses(month) {
      const classes = []

      if (!month.selectable) {
        classes.push('is-disabled')
      }
      if (this.values.from && this.values.to && this.isWithinRangeCustom(startOfMonth(month.date), endOfMonth(month.date), this.values.from, this.values.to)) {
        classes.push('is-selected')
      }
      if (this.hoverRange.length === 2 && isWithinRange(month.date, this.hoverRange[0], this.hoverRange[1])) {
        classes.push('is-in-range')
      }
      return classes
    }

    quarterClasses(quarter) {
      const classes = []

      if (!quarter.selectable) {
        classes.push('is-disabled')
      }
      if (this.values.from && this.values.to && this.isWithinRangeCustom(quarter.range.start, quarter.range.end, this.values.from, this.values.to)) {
          classes.push('is-selected')
      }
      if (this.hoverRange.length === 2 && isWithinRange(quarter.months[1].date, this.hoverRange[0], this.hoverRange[1])) {
        classes.push('is-in-range')
      }
      return classes
    }

    yearClasses(year) {
      const classes = []

      if (!year.selectable) {
        classes.push('is-disabled')
      }
      if (this.values.from && this.values.to && this.isWithinRangeCustom(startOfYear(year.date), endOfYear(year.date), this.values.from, this.values.to)) {
        classes.push('is-selected')
      }
      if (this.hoverRange.length === 2 && isWithinRange(year.date, this.hoverRange[0], this.hoverRange[1])) {
        classes.push('is-in-range')
      }
      return classes
    }

    isDateAllowed(date) {
      let isAllowed = true

      if (this.allowFrom) {
        isAllowed = isAllowed && !isBefore(date, parse(this.allowFrom))
      }
      if (this.allowTo) {
        isAllowed = isAllowed && !isAfter(date, parse(this.allowTo))
      }
      return isAllowed
    }

    isWithinRangeCustom(from, to, start, end) {
      return isWithinRange(from, start, end) || isWithinRange(to, start, end) || (isBefore(from, start) && isAfter(to, end))
    }

    isRangeAllowed([from, to]) {
      let rangeFrom
      let rangeTo

      if (!this.past) {
        rangeFrom = subDays(this.now, 1)
      } else if (this.allowFrom) {
        rangeFrom = this.allowFrom
      } else {
        rangeFrom = this.start
      }
      if (!this.future) {
        rangeTo = addDays(this.now, 1)
      } else if (this.allowTo) {
        rangeTo = this.allowTo
      } else {
        rangeFrom = this.end
      }
      return isWithinRange(from, rangeFrom, rangeTo) || isWithinRange(to, rangeFrom, rangeTo) || (isBefore(from, rangeFrom) && isAfter(to, rangeTo))
    }

    getAllowedDatesOfRange([from, to]) {
      const distance = differenceInDays(to, from)

      return new Array(distance + 1)
        .fill(null)
        .map((_, index) => addDays(from, index))
        .filter(this.isDateAllowed)
    }
  }
</script>

<style lang="scss">
.mj-daterange-picker  {
  text-align: left;
  min-width: 400px;
  width: var(--default-width);
  user-select: none;
  border: 1px solid var(--border-color);
  border-radius: 4px;

  & * {
    box-sizing: border-box;
  }
}

.mj-daterange-picker .panels-choices {
  display: grid;
  grid-gap: 10px 10px;
  grid-template-columns: repeat(auto-fit, minmax(80px, 1fr));
  border-bottom: 1px solid var(--border-color);
  padding: 20px;

  .panel-button {
    font-size: 12px;
    font-weight: bold;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    height: 32px;
    padding: 5px 15px;
    border-radius: 4px;
    cursor: pointer;
    background-color: #F2F4F5;

    &.is-current,
    &:hover {
      background-color: var(--primary-color);
    }
  }
}

.mj-daterange-picker .preset-ranges {
  padding: 20px;
  display: flex;
  flex-wrap: wrap;
  border-bottom: 1px solid var(--border-color);

  .preset {
    width: 50%;
    font-size: 13px;
    height: 20px;
    cursor: pointer;
    position: relative;
    margin: 5px 0;

    input {
      position: absolute;
      opacity: 0;
      height: 0;
      width: 0;

      &:checked ~ label .check {
        background-color: var(--primary-color);

        &::after {
          background-color: transparent;
        }
      }
    }

    label {
      display: inline-flex;
      align-items: center;

      span + span {
        margin-left: 10px;
      }

      .check {
        display: block;
        position: relative;
        height: 20px;
        width: 20px;
        background-color: var(--secondary-color);
        border-radius: 10px;

        &::after {
          content: '';
          position: absolute;
          height: 10px;
          width: 10px;
          left: 50%;
          top: 50%;
          background-color: white;
          border-radius: 100%;
          border: 3px solid white;
          transform: translateX(-50%) translateY(-50%);
        }

      }
    }

    * {
      cursor: pointer;
    }
  }
}

.mj-calendar {
  color: var(--contrast-color);
  background-color: var(--normal-color);
  padding: 20px;

  .calendar-header {
    display: flex;
    justify-content: space-between;
    align-items: center;

    .calendar-month-name {
      flex: 1;
      text-align: center;
      color: var(--secondary-color);
      font-weight: bold;
      font-size: 14px;
    }

    .calendar-arrow {
      fill: var(--secondary-color);
      cursor: pointer;
    }
  }

  .calendar-months {
    margin-top: 20px;
    display: grid;
    grid-gap: 10px 10px;
    grid-template-columns: 1fr 1fr 1fr;

    .month {
      height: 50px;
      padding: 10px;
      display: inline-flex;
      align-items: center;
      justify-content: center;
      border: 1px solid var(--border-color);
      border-radius: 4px;
      font-size: 13px;

      &:hover {
        background-color: var(--hover-range-color);
      }

      &.is-disabled {
        cursor: not-allowed;
        opacity: 0.33;
        pointer-events: none;
      }

      &.is-in-range {
        background-color: var(--hover-range-color);
      }

      &.is-selected {
        background-color: var(--primary-color);
      }

      &:not(.is-disabled) {
        cursor: pointer;
      }
    }
  }

  .calendar-quarters {
    margin-top: 20px;

    .quarter {
      display: grid;
      grid-gap: 10px 10px;
      grid-template-columns: 1fr 3fr;
      margin: 10px 0;
      align-items: center;
      font-size: 13px;

      .months {
        display: grid;
        grid-gap: 10px 10px;
        align-items: center;
        grid-template-columns: 1fr 1fr 1fr;
        border: 1px solid var(--border-color);
        border-radius: 4px;
        height: 50px;
        padding: 10px 30px;

        &:hover {
          background-color: var(--hover-range-color);
        }

        .month {
          text-align: center;
        }
      }

      &.is-disabled {
        cursor: not-allowed;
        pointer-events: none;

        .months,
        .legend {
          opacity: 0.33;
        }
      }

      &.is-in-range {
        .months {
          background-color: var(--hover-range-color);
        }
      }

      &.is-selected .months {
        background-color: var(--primary-color);
      }

      &:not(.is-disabled) .months {
        cursor: pointer;
      }
    }
  }

  .calendar-years {
    .year {
      height: 50px;
      padding: 10px;
      display: flex;
      align-items: center;
      justify-content: center;
      border: 1px solid var(--border-color);
      border-radius: 4px;
      font-size: 13px;
      margin: 10px 0;

      &:hover {
        background-color: var(--hover-range-color);
      }

      &.is-disabled {
        cursor: not-allowed;
        opacity: 0.33;
        pointer-events: none;
      }

      &.is-in-range {
        background-color: var(--hover-range-color);
      }

      &.is-selected {
        background-color: var(--primary-color);
      }

      &:not(.is-disabled) {
        cursor: pointer;
      }
    }
  }

  .calendar-days-name,
  .calendar-days {
    display: flex;
    flex-wrap: wrap;

    .day {
      width: calc(100% / 7);
      display: inline-flex;
      align-items: center;
      justify-content: center;
    }
  }

  .calendar-days-name .day {
    font-size: 11px;
    color: var(--ternary-color);
    height: 30px;
  }

  .calendar-days .day {
    height: 40px;
    font-size: 13px;
    border-top: 2px solid white;
    border-bottom: 2px solid white;

    &:not(.is-current-month) {
      color: var(--ternary-color);
    }

    &.is-disabled {
      cursor: not-allowed;
      opacity: 0.33;
      pointer-events: none;
    }

    &.is-today {
      span {
        color: var(--secondary-color);
        font-weight: bold;
      }
    }

    &.is-in-range {
      background-color: var(--hover-range-color);
    }

    &.is-first-range {
      border-top-left-radius: 4px;
      border-bottom-left-radius: 4px;
    }

    &.is-last-range {
      border-top-right-radius: 4px;
      border-bottom-right-radius: 4px;
    }

    &.is-edge-range {
      background-color: var(--primary-color);
    }

    &.is-selected {
      background-color: var(--primary-color);
    }

    &:not(.is-disabled) {
      cursor: pointer;
    }
  }
}

.mj-calendar.mj-calendar-days {
  .calendar-days .day {
    &:not(.is-edge-range):hover {
      background-color: var(--hover-day-color);
    }
  }
}

.mj-daterange-picker-controls {
  margin-top: 10px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  border-top: 1px solid var(--border-color);
  padding: 20px;

  .mj-daterange-picker-button {
    height: 36px;
    min-width: 150px;
    padding: 5px;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    border-radius: 4px;
    font-size: 12px;

    &:not(.is-disabled) {
      cursor: pointer;
    }
  }
}

.mj-daterange-picker-reset {
  border: 1px solid #E6EAED;
}

.mj-daterange-picker-submit {
  background-color: var(--primary-color);
  color: white;

  &.is-disabled {
    opacity: 0.33;
    cursor: not-allowed;
    pointer-events: none;
  }
}
</style>

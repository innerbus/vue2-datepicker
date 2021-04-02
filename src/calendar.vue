<template>
  <div :class="['mx-calendar', 'mx-calendar-' + type]">
    <div class="mx-calendar-header">
      <a
        v-show="panel !== 'TIME'"
        class="mx-icon-last-year"
        @click="handleIconYear(-1)">&laquo;</a>
      <a
        v-show="panel === 'DATE' || panel === 'WEEK'"
        class="mx-icon-last-month"
        @click="handleIconMonth(-1)">&lsaquo;</a>
      <a
        v-show="panel !== 'TIME'"
        class="mx-icon-next-year"
        @click="handleIconYear(1)">&raquo;</a>
      <a
        v-show="panel === 'DATE' || panel === 'WEEK'"
        class="mx-icon-next-month"
        @click="handleIconMonth(1)">&rsaquo;</a>
      <a
        v-show="panel === 'DATE' || panel === 'WEEK'"
        class="mx-current-month"
        @click="handleBtnMonth">{{months[calendarMonth]}}</a>
      <a
        v-show="panel === 'DATE' || panel === 'WEEK' || panel === 'MONTH'"
        class="mx-current-year"
        @click="handleBtnYear">{{calendarYearTitle}}</a>
      <a
        v-show="panel === 'YEAR'"
        class="mx-current-year">{{yearHeader}}</a>
      <a
        v-show="panel === 'TIME'"
        class="mx-time-header"
        @click="handleTimeHeader">{{timeHeader}}</a>
    </div>
    <div class="mx-calendar-content">
      <panel-date
        v-show="panel === 'DATE' || panel === 'WEEK'"
        :value="value"
        :date-format="dateFormat"
        :calendar-month="calendarMonth"
        :calendar-year="calendarYear"
        :start-at="startAt"
        :end-at="endAt"
        :first-day-of-week="firstDayOfWeek"
        :disabled-date="isDisabledDate"
        @select="selectDate"/>
      <panel-year
        v-show="panel === 'YEAR'"
        :value="value"
        :disabled-year="isDisabledYear"
        :first-year="firstYear"
        @select="selectYear" />
      <panel-month
        v-show="panel === 'MONTH'"
        :value="value"
        :disabled-month="isDisabledMonth"
        :calendar-year="calendarYear"
        @select="selectMonth" />
      <panel-time
        v-show="panel === 'TIME'"
        :minute-step="minuteStep"
        :time-picker-options="timePickerOptions"
        :value="value"
        :disabled-time="isDisabledTime"
        :time-type="timeType"
        @select="selectTime"
        @pick="pickTime" />
    </div>
  </div>
</template>

<script>
import { isValidDate, isDateObejct, formatDate } from '@/utils/index'
import locale from '@/mixins/locale'
import scrollIntoView from '@/utils/scroll-into-view'
import PanelDate from '@/panel/date'
import PanelYear from '@/panel/year'
import PanelMonth from '@/panel/month'
import PanelTime from '@/panel/time'
import moment from 'moment'

export default {
  name: 'CalendarPanel',
  components: { PanelDate, PanelYear, PanelMonth, PanelTime },
  mixins: [locale],
  props: {
    value: {
      default: null,
      validator: function (val) {
        return val === null || isValidDate(val)
      }
    },
    startAt: null,
    endAt: null,
    visible: {
      type: Boolean,
      default: false
    },
    type: {
      type: String,
      default: 'date' // ['date', 'datetime'] zendy added 'month', 'year', mxie added "time"
    },
    dateFormat: {
      type: String,
      default: 'YYYY-MM-DD'
    },
    // below user set
    firstDayOfWeek: {
      default: 7,
      type: Number,
      validator: val => val >= 1 && val <= 7
    },
    notBefore: {
      default: null,
      validator: function (val) {
        return !val || isValidDate(val)
      }
    },
    notAfter: {
      default: null,
      validator: function (val) {
        return !val || isValidDate(val)
      }
    },
    disabledDays: {
      type: [Array, Function],
      default: function () {
        return []
      }
    },
    minuteStep: {
      type: Number,
      default: 0,
      validator: val => val >= 0 && val <= 60
    },
    timePickerOptions: {
      type: [Object, Function],
      default () {
        return null
      }
    }
  },
  data () {
    const now = new Date()
    const calendarYear = now.getFullYear()
    const calendarMonth = now.getMonth()
    const firstYear = Math.floor(calendarYear / 10) * 10
    return {
      panel: 'NONE',
      dates: [],
      calendarMonth,
      calendarYear,
      firstYear
    }
  },
  computed: {
    calendarYearTitle() {
      let firstYearDate = new Date();
      firstYearDate.setFullYear(+this.calendarYear)
      const yearFormat = this.t('yearFormat') || 'yyyy';
      return moment(firstYearDate).format(yearFormat)
    },
    now: {
      get () {
        return new Date(this.calendarYear, this.calendarMonth).getTime()
      },
      set (val) {
        const now = new Date(val)
        this.calendarYear = now.getFullYear()
        this.calendarMonth = now.getMonth()
      }
    },
    timeType () {
      const h = /h+/.test(this.$parent.format) ? '12' : '24'
      const a = /A/.test(this.$parent.format) ? 'A' : 'a'
      return [h, a]
    },
    timeHeader () {
      if (this.type === 'time') {
        return this.$parent.format
      }
      return this.value && formatDate(this.value, this.dateFormat)
    },
    yearHeader () {
      let firstYearDate = new Date();
      let lastYearDate = new Date();
      firstYearDate.setFullYear(+this.firstYear)
      lastYearDate.setFullYear(+this.firstYear + 10)
      const yearFormat = this.t('yearFormat') || 'yyyy';
      return moment(firstYearDate).format(yearFormat) + ' ~ ' + moment(lastYearDate).format(yearFormat)
    },
    months () {
      return this.t('months')
    },
    notBeforeTime () {
      if (this.type === 'week') {
        // If type is week, need to add just notBefore & notAfter out of the week range
        if (this.notBefore) {
          let notBefore = this.getWeekStartDate(this.notBefore)
          return this.getCriticalTime(notBefore)
        }
      }
      return this.getCriticalTime(this.notBefore)
    },
    notAfterTime () {
      if (this.type === 'week') {
        // If type is week, need to add just notBefore & notAfter out of the week range
        if (this.notAfter) {
          let notAfter = this.getWeekStartDate(this.notAfter)
          notAfter.setDate(notAfter.getDate() + 6)
          return this.getCriticalTime(notAfter)
        }
      }
      return this.getCriticalTime(this.notAfter)
    }
  },
  watch: {
    value: {
      immediate: true,
      handler: 'updateNow'
    },
    visible: {
      immediate: true,
      handler: 'init'
    },
    panel: {
      handler: 'handelPanelChange'
    }
  },
  methods: {
    handelPanelChange (panel, oldPanel) {
      this.$parent.$emit('panel-change', panel, oldPanel)
      if (panel === 'YEAR') {
        this.firstYear = Math.floor(this.calendarYear / 10) * 10
      } else if (panel === 'TIME') {
        this.$nextTick(() => {
          const list = this.$el.querySelectorAll('.mx-panel-time .mx-time-list')
          for (let i = 0, len = list.length; i < len; i++) {
            const el = list[i]
            scrollIntoView(el, el.querySelector('.actived'))
          }
        })
      }
    },
    init (val) {
      if (val) {
        const type = this.type
        if (type === 'month') {
          this.showPanelMonth()
        } else if (type === 'year') {
          this.showPanelYear()
        } else if (type === 'time') {
          this.showPanelTime()
        } else if (type === 'week') {
          this.showPanelWeek()
        } else {
          this.showPanelDate()
        }
      } else {
        this.showPanelNone()
      }
      this.updateNow(this.value)
    },
    // 根据value更新日历
    updateNow (value) {
      if (this.type === 'week') {
        this.now = this.startAt ? this.startAt : new Date()
      } else {
        this.now = value ? new Date(value) : new Date()
      }
    },
    getCriticalTime (value) {
      if (!value) {
        return null
      }
      const date = new Date(value)
      if (this.type === 'year') {
        return new Date(date.getFullYear(), 0).getTime()
      } else if (this.type === 'month') {
        return new Date(date.getFullYear(), date.getMonth()).getTime()
      } else if (this.type === 'date') {
        return date.setHours(0, 0, 0, 0)
      }
      return date.getTime()
    },
    inBefore (time, startAt) {
      startAt = this.type === 'week' ? undefined : startAt || this.startAt
      return (this.notBeforeTime && time < this.notBeforeTime) ||
        (startAt && time < this.getCriticalTime(startAt))
    },
    inAfter (time, endAt) {
      endAt = this.type === 'week' ? undefined : endAt || this.endAt
      return (this.notAfterTime && time > this.notAfterTime) ||
        (endAt && time > this.getCriticalTime(endAt))
    },
    inDisabledDays (time) {
      if (Array.isArray(this.disabledDays)) {
        return this.disabledDays.some(v => this.getCriticalTime(v) === time)
      } else if (typeof this.disabledDays === 'function') {
        return this.disabledDays(new Date(time))
      }
      return false
    },
    isDisabledYear (year) {
      const time = new Date(year, 0).getTime()
      const maxTime = new Date(year + 1, 0).getTime() - 1
      return this.inBefore(maxTime) || this.inAfter(time) || (this.type === 'year' && this.inDisabledDays(time))
    },
    isDisabledMonth (month) {
      const time = new Date(this.calendarYear, month).getTime()
      const maxTime = new Date(this.calendarYear, month + 1).getTime() - 1
      return this.inBefore(maxTime) || this.inAfter(time) || (this.type === 'month' && this.inDisabledDays(time))
    },
    isDisabledDate (date) {
      const time = new Date(date).getTime()
      const maxTime = new Date(date).setHours(23, 59, 59, 999)
      return this.inBefore(maxTime) || this.inAfter(time) || this.inDisabledDays(time)
    },
    isDisabledTime (date, startAt, endAt) {
      const time = new Date(date).getTime()
      return this.inBefore(time, startAt) || this.inAfter(time, endAt) || this.inDisabledDays(time)
    },
    selectDate (date) {
      if (this.type === 'datetime') {
        let time = new Date(date)
        if (isDateObejct(this.value)) {
          time.setHours(
            this.value.getHours(),
            this.value.getMinutes(),
            this.value.getSeconds()
          )
        }
        if (this.isDisabledTime(time)) {
          time.setHours(0, 0, 0, 0)
          if (this.notBefore && time.getTime() < new Date(this.notBefore).getTime()) {
            time = new Date(this.notBefore)
          }
          if (this.startAt && time.getTime() < new Date(this.startAt).getTime()) {
            time = new Date(this.startAt)
          }
        }
        this.selectTime(time)
        this.showPanelTime()
        return
      } else if (this.type === 'week') {
        const startDate = this.getWeekStartDate(date)
        const endDate = new Date(startDate)
        endDate.setDate(startDate.getDate() + 6)

        this.$emit('select-week', [startDate, endDate])
        return
      }
      this.$emit('select-date', date)
    },
    getWeekStartDate (date) {
      const startDate = new Date(date)
      // First day of the week: JS: 0-6 (Sun - Sat), DatePicker: 1-7 (Mon-Sun)
      startDate.setHours(0, 0, 0, 0)
      startDate.setDate(startDate.getDate() - startDate.getDay() + (this.firstDayOfWeek < 7 ? this.firstDayOfWeek : this.firstDayOfWeek - 7))
      return startDate
    },
    selectYear (year) {
      this.changeCalendarYear(year)
      if (this.type.toLowerCase() === 'year') {
        return this.selectDate(new Date(this.now))
      }
      this.showPanelMonth()
    },
    selectMonth (month) {
      this.changeCalendarMonth(month)
      if (this.type.toLowerCase() === 'month') {
        return this.selectDate(new Date(this.now))
      }
      this.showPanelDate()
    },
    selectTime (time) {
      this.$emit('select-time', time, false)
    },
    pickTime (time) {
      this.$emit('select-time', time, true)
    },
    changeCalendarYear (year) {
      this.now = new Date(year, this.calendarMonth)
    },
    changeCalendarMonth (month) {
      this.now = new Date(this.calendarYear, month)
    },
    getSibling () {
      const calendars = this.$parent.$children.filter(v => v.$options.name === this.$options.name)
      const index = calendars.indexOf(this)
      const sibling = calendars[index ^ 1]
      return sibling
    },
    handleIconMonth (flag) {
      const month = this.calendarMonth
      this.changeCalendarMonth(month + flag)
      this.$parent.$emit('change-calendar-month', {
        month,
        flag,
        vm: this,
        sibling: this.getSibling()
      })
    },
    handleIconYear (flag) {
      if (this.panel === 'YEAR') {
        this.changePanelYears(flag)
      } else {
        const year = this.calendarYear
        this.changeCalendarYear(year + flag)
        this.$parent.$emit('change-calendar-year', {
          year,
          flag,
          vm: this,
          sibling: this.getSibling()
        })
      }
    },
    handleBtnYear () {
      this.showPanelYear()
    },
    handleBtnMonth () {
      this.showPanelMonth()
    },
    handleTimeHeader () {
      if (this.type === 'time') {
        return
      }
      this.showPanelDate()
    },
    changePanelYears (flag) {
      this.firstYear = this.firstYear + flag * 10
    },
    showPanelNone () {
      this.panel = 'NONE'
    },
    showPanelTime () {
      this.panel = 'TIME'
    },
    showPanelDate () {
      this.panel = 'DATE'
    },
    showPanelWeek () {
      this.panel = 'WEEK'
    },
    showPanelYear () {
      this.panel = 'YEAR'
    },
    showPanelMonth () {
      this.panel = 'MONTH'
    }
  }
}
</script>

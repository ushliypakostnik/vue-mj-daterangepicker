<template lang="pug">
  div
    div(style="margin: 20px 0;")
      date-range-picker(:yearsCount="3" locale="ru" @select="checkUpdate" :show-controls="false" :presets="[]" :range="true")

    div(style="margin: 20px 0;")
      date-range-picker(:yearsCount="3" locale="ru" @select="checkUpdate" :show-controls="false" :range="false")
</template>

<script lang="ts">
  import { Component, Vue } from 'vue-property-decorator'
  import DateRangePicker from '@/components/DateRangePicker.vue'

  @Component({
    components: {
      DateRangePicker
    },
  })

  export default class Home extends Vue {
    checkUpdate(values) {
      // console.log(values);
    }
  }
</script>

<style>
.button {
    appearance: none;
    border: 0;
    display: inline-block;
    font-size: 12px;
    font-weight: bold;
    height: 32px;
    padding: 5px 15px;
    border-radius: 4px;
    cursor: pointer;
    background-color: #F2F4F5;
}

.button + .button {
    margin-left: .25em;
}

.button.selected {
    background-color: #3297DB;
    color: #F2F4F5;
}
</style>

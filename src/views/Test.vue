<template lang="pug">
  div
    div(style="margin: 20px 0;")
      date-range-picker(:yearsCount="3" locale="ru" @select="checkUpdate" :show-controls="false" :theme="theme" :presets="[]" :range="true")

    div(style="margin: 20px 0;")
      date-range-picker(:yearsCount="3" locale="ru" @select="checkUpdate" :show-controls="false" :theme="theme" :range="false")
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
    theme = {
      primary: '#46C3A3',
      secondary: '#2D3E50',
      ternary: '#93A0BD',
      light: '#ffffff',
      border: '#e6e6e6',
      dark: '#000000',
      hovers: {
        day: '#e6e6e6',
        range: '#e6e6e6'
      }
    }

    get now() {
      return new Date().toISOString()
    }

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

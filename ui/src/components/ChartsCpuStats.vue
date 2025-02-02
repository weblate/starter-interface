<template>
  <LineChart
    :chart-data="chartData"
    :options="options"
    :height="$q.screen.gt.xs ? 150 : 400"
  />
</template>

<script lang="ts">
import { AxiosResponse } from 'axios'
import { expressApi } from 'boot/axios'
import {
  CategoryScale,
  Chart,
  ChartData,
  ChartOptions,
  LinearScale,
  LineController,
  LineElement,
  PointElement,
  Title
} from 'chart.js'
import { colors, getCssVar, LoadingBar } from 'quasar'
import { defineComponent, onMounted, onUnmounted, ref } from 'vue'
import { LineChart } from 'vue-chart-3'
import { useI18n } from 'vue-i18n'

interface CpuStatReq {
  data: {
    currentLoad: number
    cpus: Array<{ load: string }>
  }
}

Chart.register(
  CategoryScale,
  LineController,
  LineElement,
  LinearScale,
  PointElement,
  Title
)

export default defineComponent({
  name: 'ChartsCpuStats',
  components: { LineChart },
  props: {
    maxDataPoints: {
      type: Number,
      default: 20
    },
    pollInterval: {
      type: Number,
      default: 1500
    }
  },

  setup(props) {
    // eslint-disable-next-line @typescript-eslint/unbound-method
    const { t } = useI18n()

    // Set IDs of the systeminfo service required
    const systeminfoCmdCurrentLoad = 'l'

    const chartBackgroundOpacity = 70
    const isUnMounted = ref<boolean>(false)

    onMounted(() => {
      // Disables the AJAX loading bar at the bottom of the page for polled system info requests
      // to avoid it flashing on each poll
      LoadingBar.setDefaults({
        hijackFilter(url: string) {
          return !url.includes('systeminfo')
        }
      })

      // Start the polling
      void fetchCpuStats()
    })

    onUnmounted(() => {
      // Re-enables the AJAX loading bar at the bottom of the page for system info requests
      LoadingBar.setDefaults({
        hijackFilter() {
          return true
        }
      })

      // Instructs to exit the running poll loop, because this component is no longer visible
      isUnMounted.value = true
    })

    const chartData = ref<ChartData<'line'>>({
      labels: [0],
      datasets: [
        {
          backgroundColor: colors.lighten(
            getCssVar('secondary') as string,
            chartBackgroundOpacity
          ),
          borderWidth: 2,
          fill: 'start',
          data: [0]
        }
      ]
    })

    const options = ref<ChartOptions<'line'>>({
      animation: {
        duration: 0
      },
      elements: {
        point: {
          radius: 0
        }
      },
      responsive: true,
      maintainAspectRatio: false,
      scales: {
        yAxis: {
          min: 0,
          max: 100,
          display: true
        },
        xAxis: {
          display: false
        }
      },

      plugins: {
        title: {
          display: true,
          align: 'start',
          text: t('components.charts.cpu_stats.cpu_status'),
          padding: {
            top: 10,
            bottom: 20
          }
        },
        tooltip: {
          enabled: false
        },
        legend: {
          display: false
        }
      }
    })

    function delay(ms: number) {
      return new Promise((resolve) => {
        setTimeout(resolve, ms)
      })
    }

    async function fetchCpuStats(): Promise<void> {
      try {
        const cpuStat: AxiosResponse<CpuStatReq> = await expressApi.post(
          '/v1/system/systeminfo',
          { id: systeminfoCmdCurrentLoad }
        )

        // If there are more than X items in the object, remove one
        // to avoid it growing too big
        if (chartData.value.datasets[0].data.length > props.maxDataPoints) {
          chartData?.value?.labels?.shift()
          chartData.value.datasets[0].data.shift()
        }

        // Add the fetched data in to the chart
        chartData?.value?.labels?.push(
          new Date(new Date().getTime()).toLocaleTimeString()
        )
        chartData.value.datasets[0].data.push(cpuStat.data.data.currentLoad)

        if (cpuStat.data.data.currentLoad > 50) {
          chartData.value.datasets[0].backgroundColor = colors.lighten(
            getCssVar('warning') as string,
            chartBackgroundOpacity
          )
          chartData.value.datasets[0].borderColor = getCssVar(
            'warning'
          ) as string
        } else if (cpuStat.data.data.currentLoad > 80) {
          chartData.value.datasets[0].backgroundColor = colors.lighten(
            getCssVar('negative') as string,
            chartBackgroundOpacity
          )
          chartData.value.datasets[0].borderColor = getCssVar(
            'negative'
          ) as string
        } else {
          chartData.value.datasets[0].backgroundColor = colors.lighten(
            getCssVar('secondary') as string,
            chartBackgroundOpacity
          )
          chartData.value.datasets[0].borderColor = getCssVar(
            'secondary'
          ) as string
        }
      } catch (error) {
        // Continuing as may have just been due to a temporary connection issue
        console.error('Error fetching CPU stats')
        console.error(error)
      }

      if (isUnMounted.value) {
        // Stop the loop and reset chart content
        chartData.value.datasets = []
      } else {
        // Delay before calling next fetch
        await delay(props.pollInterval)
        void fetchCpuStats()
      }
    }

    return {
      chartData,
      options,
      props
    }
  }
})
</script>
